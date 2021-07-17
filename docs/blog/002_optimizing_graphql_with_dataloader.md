# Optimizing GraphQL with Dataloader

`2020-04-01`

(First appeared on the [ErlangSolutions blog](https://www.erlang-solutions.com/blog/optimizing-graphql-with-dataloader.html).)

I spent the last fourteen months working with a client on mobility applications. We developed several parts of the backend of a corporate car sharing platform and built an application from scratch to conduct and analyze test drives at dealerships around the World. Development of the former was already started before we got there, so the tech stack was pretty much decided. We developed a handful of services exposing GraphQL APIs. I think [Elixir](https://elixir-lang.org/) was the right fit for this purpose. Using the [Absinthe](https://hexdocs.pm/absinthe/overview.html) library to craft the APIs was a good choice. However, multiple services had to communicate with each-other and making a responsive UI requires optimized queries whenever possible. We did our best and learned from our mistakes.

When we started working on the second application, we had more flexibility in choosing the tools. We decided to stick with a GraphQL API and used Absinthe, but paid attention to how we write the resolver functions. In the end our API was blazing fast, because most of our resolvers ran only a handful of queries.

Designing a usable API is art, as is optimizing the backend that serves that API. In this blog post - and hopefully some upcoming ones - I'll give you a few hints on what to avoid or what to strive for when working with Absinthe. We cannot always affect how other applications/services work, but we can do our best to make our service as fast as possible.

There are plenty of resources out there to help you get started with Absinthe and the number of tutorials on how to use [Dataloader](https://hexdocs.pm/dataloader/Dataloader.html) are growing, but it is a recurring topic on Slack. I hope to give you an insight on how it works and how you can make use of it in your project.

## What is Dataloader in a nutshell?

In short, Dataloader is a tool that can help your application fetch data in an optimal way by implementing batching and caching.

### Dataloader in Elixir

In the Elixir world it is provided by a [library](https://hex.pm/packages/dataloader) and integrates well with Ecto and Absinthe. Note that it's not a silver bullet, your queries using Dataloader rely on the optimal implementation of the underlying library for the specific datasource.

We are not going into details on how to use Dataloader on its own. Please consult the [official documentation](https://hexdocs.pm/dataloader/Dataloader.html) for that.

Instead, let's dive into using Dataloader in Absinthe.

## Using Dataloader in Absinthe Resolvers

For the examples we are going to use rather simple schemas in order to not get lost in the details. Assume we are building a database where we keep track of companies and their employees. One employee belongs to exactly one company, but each company may have multiple employees. The Ecto schemas may be defined as:

```elixir
schema "employees" do
  field(:name, :string)
  field(:email, :string)
  belongs_to(:company, Company)
end

schema "companies" do
  field(:name, :string)
  has_many(:employees, Employee)
end
```

And the corresponding object definitions in Absinthe GraphQL may look like:

```elixir
object(:employee) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))
  field(:email, non_null(:string))
end

object(:company) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))
end
```

This is all good until we want to resolve the employees on the company. A naïve field definition of it may be:
```elixir
object(:company) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))

  field :employees, non_null(list_of(:employee)) do
    resolve(fn company, _args, _info ->
      employees = Ecto.assoc(company, :employees) |> Repo.all()

      {:ok, employees}
    end)
  end
end
```

Similarly, we can add the field `:company` to the employee object as:
```elixir
field(:company, non_null(:company)) do
  resolve(fn employee, _args, _info ->
    company = Ecto.assoc(employee, :company) |> Repo.one()

    {:ok, company}
  end)
end
```

If we now query a company through GraphQL and also ask for the employees on that field, then our backend will perform two SQL queries. This is not bad at all, but imagine the case where we have ten employee results. Moreover, each result asks for its own company field which causes several duplicate queries to Ecto.

```graphql
query($id: ID!) {
  company(id: $id) {
    id
    name
    employees {
      id
      name
      company {
        id
        name
      }
    }
  }
}
```

This may not happen in this exact form, but it helps us imagine what happens when the same associated object has to be resolved for a list of results. We have to make several queries to Ecto for this to be answered. One query to resolve the company in the root, one query for each employee and one additional query for resolving the company of each employee. 21 queries overall, we're facing the infamous n+1 problem. This is where Dataloader comes to the rescue.

### Adding Dataloader

The documentation of [Absinthe](https://hexdocs.pm/absinthe/ecto.html#dataloader) is a good starting point for using Dataloader with an Ecto data source. In short, if we want to use Dataloader in our resolvers we have to do two things:
- Add a dataloader struct to the resolution context
- Add `Absinthe.Middleware.Dataloader` to the list of plugins in our schema

We can create a dataloader struct with `Dataloader.new/1`. After that, we'll have to add sources to dataloader with `Dataloader.add_source/3`. If we have an Ecto repository (let's call it `Repo`), we can add it as follows:

```elixir
def context(ctx) do
  loader =
    Dataloader.new()
    |> Dataloader.add_source(Repo, Dataloader.Ecto.new(Repo))

  Map.put(ctx, :loader, loader)
end
```

Let's not forget to add the Dataloader plugin:

```elixir
def plugins() do
  [Absinthe.Middleware.Dataloader | Absinthe.Plugin.defaults()]
end
```

Absinthe provides a convenient way for using Dataloader in our resolvers. We just have to import `Absinthe.Resolution.Helpers` in our schema and we can use the `dataloader/1` function to resolve fields using the `Repo` datasource. Our Absinthe object definitions become:

```elixir
object(:employee) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))
  field(:email, non_null(:string))

  field(:company, non_null(:company)) do
    resolve(dataloader(Repo))
  end
end

object(:company) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))

  field(:employees, non_null(list_of(:employee))) do
    resolve(dataloader(Repo))
  end
end
```

With these modifications, resolving the graphql query above requires only three Ecto queries. That is a significant improvement.

### The query function

One useful feature of the `Dataloader.Ecto` source is that we can pass a query function to it which can be used for filtering or processing parameters that are common to many fields, for example, pagination arguments.

```elixir
def context(ctx) do
  loader =
    Dataloader.new()
    |> Dataloader.add_source(Repo, Dataloader.Ecto.new(Repo, query: &Repo.dataloader_query/2))

  Map.put(ctx, :loader, loader)
end
```

Where we can define `Repo.dataloader_query/2` to process parameters related to pagination and also leave room for extending it easily.

```elixir
def dataloader_query(queryable, params) do
  queryable
  |> paginate(params[:paginate])
end

def paginate(query, nil), do: query

def paginate(query, params) do
  from d in query,
    limit: ^params[:limit],
    offset: ^params[:offset]
end
```

Note that so far we didn't have to write any Ecto queries, because we used the `dataloader/1` helper from `Absinthe.Resolution.Helpers`.

Unfortunately we're not done yet. Assume we added the `paginate` parameter to the `employees` field on the `company` object.

```elixir
object(:company) do
  field(:id, non_null(:string))
  field(:name, non_null(:string))

  field(:employees, non_null(list_of(:employee))) do
    arg(:paginate, :pagination_input)

    resolve(dataloader(Repo))
  end
end
```

This will work, if we only have one company, but as soon as we make two `company` queries with the same pagination paremeter (for example, `{"paginate": {"limit": 10, "offset": 0}}`) for the `employees` in one single GraphQL query, we'll see an anomaly.

```elixir
query($id1: ID!, $id2: ID!, $paginate: PaginationInput) {
  res1: company(id: $id1) {
    id
    name
    employees(paginate: $paginate) {
      id
      name
    }
  }
  res2: company(id: $id2) {
    id
    name
    employees(paginate: $paginate) {
      id
      name
    }
  }
}
```

We only see employees for one of the companies, while the list of employees for the other one is empty. This happens, because the Ecto Dataloader tries to fetch the employees for both companies with a single query that includes an `order_by` on the `company_id` field. This works well when we don't want to add any `limit` or `offset` parameters. One workaround for this is to modify the argument list, which will force the loader to make separate queries for each company.

```elixir
field(:employees, non_null(list_of(:employee))) do
  arg(:paginate, :pagination_input)

  resolve(
    dataloader(Repo, fn company, args, _info ->
      {:employees, Map.put(args, :company_id, company.id)}
    end)
  )
end
```

Here we made use of a dataloader key function that should return a resource (`:employees` in this case) and a list of arguments that are passed on to our `Repo.dataloader_query/2` function. Since `:company_id` is different for the two companies, the keys for the dataloader cache will be different. One limitation of this solution is that Dataloader will have to make separate Ecto queries for each company. If this causes performance issues, then regular resolvers or rather batch resolvers may be implemented with optimal queries.

The query function is also useful if we want to have more flexibility, for example, filtering or ordering results. For this, we'll have to extend our query function to process extra parameters. In general, I like to write query helpers that take a queryable object as their first parameter and return a queryable. For example (assuming we have a status field on the employee):

```elixir
def where_active(employee) do
  from e in employee,
    where: e.status == "active"
end
```

If we extend the query function (`Repo.dataloader_query/2`) as below, we will be able to use these helpers easily:

```elixir
def dataloader_query(queryable, params) do
  queryable
  |> paginate(params[:paginate])
  |> apply_query(params[:query], params[:query_args] || [])
end

def apply_query(queryable, nil, _query_args), do: queryable

def apply_query(queryable, query, query_args) do
  apply(query, [queryable | query_args])
end
```

And we can now resolve only active employees on companies if we specify `query` in the resolver:
```elixir
field(:employees, non_null(list_of(:employee))) do
  arg(:paginate, :pagination_input)

  resolve(
    dataloader(Repo, fn company, args, _info ->
      args =
        args
        |> Map.put(:company_id, company.id)
        |> Map.put(:query, &Blog.Dataloader.Employee.Query.where_active/1)

      {:employees, args}
    end)
  )
end
```

Of course other filtering is also possible and we can also construct our queries based on the GraphQL parameters that the resolver receives in `args`.

Note that now `:query` becomes part of the key that is used in the dataloader cache, so querying the active and non-active employees of the same company in one GraphQL query might require two database queries.

### Dataloader.KV

So far we have only seen examples on how to use `Dataloader.Ecto`. What if we need to collect some data from another service in order to respond to the GraphQL query? We can use `Dataloader.KV` to retrieve and cache data based on keys.

For this we will need a function that receives two parameters. The first parameter is a batch key that groups together different objects. We will return to this shortly. The second parameter is a list, usually a list of objects or IDs to which the required data is to be retrieved. The function should return a map where the keys are the elements of this list and the associated values are the corresponding retrieved data. For example, if we store the addresses of employees in a different service, we may write a loader function as below:

```elixir
def fetch_addresses(_batch_key, employees) do
  ids = Enum.map(employees, & &1.id)

  results = call_to_another_service(ids)

  employees
  |> Map.new(&{&1, lookup_result_for_employee(&1, results)})
end
```

This function receives a list of employees and returns a map where each employee is mapped to an address. How the call to another service and the lookup are done are implementation details, but in order for the solution to be optimal the other service should support querying data in batches (for all IDs at once instead of one-by-one).

To use this function as a load function for a `Dataloader.KV` source we may change the context function in the schema as follows.

```elixir
def context(ctx) do
  loader =
    Dataloader.new()
    |> Dataloader.add_source(Repo, Dataloader.Ecto.new(Repo, query: &Repo.dataloader_query/2))
    |> Dataloader.add_source(:address, Dataloader.KV.new(&Address.fetch_addresses/2))

  Map.put(ctx, :loader, loader)
end
```

Then we can resolve the address field for each employee using the `:address` dataloader source:

```elixir
object(:employee) do
  # ... existing fields here

  field(:address, non_null(:string)) do
    resolve(dataloader(:address))
  end
end
```

As you can see, we did not use the batch key in our loader, which means we will handle all employees in one batch. This is usually fine. Batch key can be useful, if we intend to pass on certain arguments to the other service or refine the results. Perhaps we have a user token that we intend to supply for the service to check whether the user has the required access rights:

```elixir
object(:employee) do
  # ... existing fields here

  field(:address, non_null(:string)) do
    resolve(
      dataloader(:address, fn _employee, _args, %{context: %{user_token: token}} ->
        {:address, %{user_token: token}}
      end)
    )
  end
end
```

Then make the load function handle the additional arguments:

```elixir
def fetch_addresses({:address, %{user_token: _token} = args}, employees) do
  ids = Enum.map(employees, & &1.id)
  results = call_to_another_service(ids, args)

  employees
  |> Map.new(&{&1, lookup_result_for_employee(&1, results)})
end
```

Note that for each different batch key we have to make a call to the other service, so we have to be careful when specifying the arguments. For example, if we pass in a unique ID (e.g., `employee.id`), then we lose the advantage of batching, the function is called for each employee.

In general, constructing the batch key provides flexibility, but it can also hide important details in the code. Use with caution.

### More control over dataloader

In some cases we may want to have more control over how we want to handle loading and post processing data. In `Absinthe.Resolution.Helpers` there's an `on_load/2` function that takes a dataloader struct and a callback. It is useful when we have to obtain information that relies on data that can be retrieved by dataloader. The following example is not likely to appear in a real world scenario, but it demonstrates how we can make use of the `on_load` function :

```elixir
object(:company) do
  # ... other fields here
  field(:number_of_distinct_addresses_of_employees, non_null(:integer)) do
    resolve(fn company, _args, %{context: %{loader: loader}} ->
      loader
      |> Dataloader.load(Repo, :employees, company)
      |> on_load(fn loader_with_employees ->
        employees = Dataloader.get(loader_with_employees, Repo, :employees, company)

        loader_with_employees
        |> Dataloader.load_many(:address, %{}, employees)
        |> on_load(fn loader_with_addresses ->
          addresses = Dataloader.get_many(loader_with_addresses, :address, %{}, employees)

          {:ok, length(Enum.uniq(addresses))}
        end)
      end)
    end)
  end
end
```

Notice how we took advantage of the fact that we can embed `on_load` calls to optimize fetching the results. First we tell dataloader to load the employees, then we use those employees to load their addresses. Finally, we fetch the addresses and count how many unique ones there are.

In general this kind of resolver is useful when we want to move data one (or more) level up the tree with or without aggregation. In one project I used the same solution to retrieve telematics data of vehicles to be aggregated and displayed on certain trips taken with those vehicles. Both the vehicles and telematics data needed to be queried from other services.

## Conclusion

Dataloader is a poverful tool when it comes to optimizing queries, but we have to be aware of its limitations. We saw a few simple examples to get up and running with Dataloader and Absinthe. This is only the tip of the iceberg and I am hoping to follow up with some more advanced tricks and tips.
