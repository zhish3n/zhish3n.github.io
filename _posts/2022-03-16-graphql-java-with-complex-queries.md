---
layout: post
title:  "Efficient Complex Queries in GraphQL Java"
date:   2022-03-16 18:19:25 +0800
permalink: "/efficient-complex-queries-graphql-java"
description: A tutorial on implementing complex queries and efficient practices in GraphQL Java.
tags: java graphql spring
# permalink: /:categories/:year/:month
---

### Implementing Efficient and Complex Queries in a GraphQL Java Service

In this post I set up a Spring service using GraphQL Java and show how we can implement three types of GraphQL queries, as well as several best practices for making our queries quick and efficient.
This post should be read as a continuation of the [simple GraphQL starter post](./simple-graphql-starter) and [GraphQL with databases post](/graphql-java-spring-database), as those go over basic expectations of GraphQL and in this post we'll be building off some concepts explored there.

First, use [Spring Initializr](https://start.spring.io/) to kickstart setting up a new Spring Boot project.
Use the settings Gradle, Java 8 (I am using [Amazon Corretto 8](https://docs.aws.amazon.com/corretto/latest/corretto-8-ug/downloads-list.html)), and Spring Boot 2.6.4.
In the `build.gradle` of the new project, add these dependencies:

```
implementation 'org.springdoc:springdoc-openapi-ui:1.6.6'
implementation 'com.graphql-java:graphql-java:10.0'
implementation 'com.google.guava:guava:26.0-jre'
compileOnly "org.projectlombok:lombok:1.18.16"
annotationProcessor "org.projectlombok:lombok:1.18.16"
```

- The first dependency, which we have not seen so far in the other posts, helps us define API endpoints and provides a UI for us to visualize those endpoints.
- The second dependency is the base GraphQL Java library.
- The Guava dependency allows us to manipulate data objects we will use in place of a real datastore.
- Lombok allows us to add certain annotations to reduce boilerplate code.

Set up the GraphQL schema in `src/main/resources/` called `schema.graphqls`.
It should contain the following code.

```
schema {
    query: Query
}

type Query {
    parentById(id: ID): Parent
    parentBySecondaryId(secondaryId: ID): [Parent]
    parentByMultiField(where: ParentSearch): [Parent]
    childById(id: ID): Child
}

type Parent {
    id: ID
    name: String
    secondaryId: ID
    childId: Child
}

type Child {
    id: ID
    name: String
}

input ParentSearch {
    id: ID
    name: String
    secondaryID: ID
    childId: ID
}
```

In this schema we're defining four different base queries (although we are going to start with just implementing the `getById` query):

- `parentById`, which takes in an `id` and returns a single `Parent` object
- `parentBySecondaryId`, which takes in an `id` and returns any number of parents where their `secondaryId` field matches with `id`
- `parentByMultiField`, which takes in any combination of fields and values and returns any number of matching parents
- `childById`, which is the same as `parentById` but returns `Child` objects

In the previous two posts, we've implemented (pretty much) identical base queries to `parentById` and `childById`.
However, here we're going to be looking at implementing two new types of queries: a query that returns a list of objects matching one identifier, and a query that returns a list of objects matching any number of identifiers.
Note how we're constructing the multi-field query in the schema:
instead of passing it a field with a defined value type, we're actually passing it a new `input ParentSearch`, which defines the possible fields we can pass the base query.

Let's set up our data models.
Create a folder called `model` in `src/main/java/`.
In it, create two classes called `Parent` and `Child`.
They should look like this:

```java
@Builder
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Parent {

    private UUID id;
    private String name;
    private UUID secondaryId;
    private UUID childId;

}
```

```java
@Builder
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Child {

    private UUID id;
    private String name;

}
```

We won't be using an H2 datastore here, but we'll still be doing something a little more complex than a floating list of maps.
Create a folder called `data` in `src/main/java/` and create a new class inside it called `DummyData`.

```java
public class DummyData {

    static UUID dummySecondaryId = UUID.fromString("453b631f-b068-4630-bda4-930571e689c2");

    static Parent p1 = new Parent(
            UUID.fromString("92bac69b-ffa1-4cb5-941f-fe1b9c4115da"),
            "parent1",
            dummySecondaryId,
            UUID.fromString("e84a9d26-3350-4c7e-8586-69317262e7e0")
    );

    static Parent p2 = new Parent(
            UUID.fromString("9cdc7680-0201-48d5-a53b-3244a9bd4928"),
            "parent2",
            dummySecondaryId,
            UUID.fromString("fafe1b57-32bc-43eb-a7b4-41aa05bec16c")
    );

    static Map<String, Parent> parentData = new LinkedHashMap<>();

    static {
        parentData.put("92bac69b-ffa1-4cb5-941f-fe1b9c4115da", p1);
        parentData.put("9cdc7680-0201-48d5-a53b-3244a9bd4928", p2);
    }

    static Child c1 = new Child(
            UUID.fromString("e84a9d26-3350-4c7e-8586-69317262e7e0"),
            "child1"
    );

    static Child c2 = new Child(
            UUID.fromString("fafe1b57-32bc-43eb-a7b4-41aa05bec16c"),
            "child2"
    );

    static Map<String, Child> childData = new LinkedHashMap<>();

    static {
        childData.put("e84a9d26-3350-4c7e-8586-69317262e7e0", c1);
        childData.put("fafe1b57-32bc-43eb-a7b4-41aa05bec16c", c2);
    }
    
    public static Object getEntityData(String id) {
        if (parentData.get(id) != null) {
            return parentData.get(id);
        } else if (childData.get(id) != null) {
            return childData.get(id);
        }
        return null;
    }
}
```

Imagine this class as a datastore.
We've inserted two `Parent` rows and two `Child` rows.
We've also defined a method `getEntityData(id)` that can be thought of as an SQL query along the lines of `select 1 from any where id = x`.
This method will be used by our two base queries `parentById` and `childById`.

Next, create a class called `GraphQLProvider` in `src/main/java/provider/` with the following content:

```java
@Service
public class GraphQLProvider {

  private GraphQL graphQL;
  private DataLoaderRegistry dataLoaderRegistry;
  private EntityWiring entityWiring;

  @Autowired
  public GraphQLProvider(DataLoaderRegistry dataLoaderRegistry, EntityWiring entityWiring) {
    this.dataLoaderRegistry = dataLoaderRegistry;
    this.entityWiring = entityWiring;
  }

  @PostConstruct
  public void init() throws IOException {
    URL url = Resources.getResource("schema.graphqls");
    String sdl = Resources.toString(url, Charsets.UTF_8);
    GraphQLSchema graphQLSchema = buildSchema(sdl);
    DataLoaderDispatcherInstrumentation dlInstrumentation =
        new DataLoaderDispatcherInstrumentation(dataLoaderRegistry, newOptions().includeStatistics(true));
    Instrumentation instrumentation = new ChainedInstrumentation(
        asList(new TracingInstrumentation(), dlInstrumentation)
    );
    this.graphQL = GraphQL.newGraphQL(graphQLSchema).instrumentation(instrumentation).build();
  }

  private GraphQLSchema buildSchema(String sdl) {
    TypeDefinitionRegistry typeRegistry = new SchemaParser().parse(sdl);
    RuntimeWiring runtimeWiring = buildWiring();
    SchemaGenerator schemaGenerator = new SchemaGenerator();
    return schemaGenerator.makeExecutableSchema(typeRegistry, runtimeWiring);
  }

  private RuntimeWiring buildWiring() {
    return RuntimeWiring.newRuntimeWiring()
        .type(newTypeWiring("Query") // Defines base queries that can be made
            .dataFetcher("parentById", entityWiring.xByIdDataFetcher)
            .dataFetcher("childById", entityWiring.xByIdDataFetcher)
        )
        .type(newTypeWiring("Parent") // Teaches type Parent how to get Child
            .dataFetcher("childId", entityWiring.getChildFromParentDataFetcher)
        )
        .build();
  }

  @Bean
  public GraphQL graphQL() {
    return graphQL;
  }

}
```

Notice that this file looks quite similar to the `GraphQLService` class we made in the [simple GraphQL starter post](./simple-graphql-starter).
We're still grabbing the schema from `resources` and compiling it with the wiring we define in `buildWiring`.
However, now instead of `@Autowire`-ing in resolvers, we're bringing in the use of `DataLoader`s.

In essence, `DataLoader`s help cache data across similar queries. 
Using them allows our service to run much more efficiently, especially when it comes to queries that end up running the same resolver methods over and over (think of `people` objects with overlapping `friends`).
You can read more about the GraphQL `DataLoader` [here](https://www.graphql-java.com/documentation/batching/) and [here](https://xuorig.medium.com/the-graphql-dataloader-pattern-visualized-3064a00f319f).
Incorporating `DataLoaders` into our service makes it so that when resolvers want to fetch data, they don't go straight to a datastore;
instead, they pass identifiers to relevant `loaders`, which then aggregate similar identifiers to batch load more efficiently.

In the class above, this is represented by the use of `DataLoaderRegistry` and `EntityWiring`.
The `DataLoaderRegistry` holds a map of unique data loader keys to `DataLoader` objects that handle the cache for an output type.
We add it to our `GraphQL` service through the use of `DataLoaderDispatcherInstrumentation`.
Here, we also chain a `TracingInstrumentation` which will allow us to review `DataLoader` performance.
`EntityWiring`, which we haven't created yet, is where we define:

- Data-getting methods (e.g. HTTP call or query from some datastore)
- `BatchLoader`s that asynchronously call those methods
- `DataLoader`s utilizing those `BatchLoader`s
- The `DataLoaderRegistry` registrations between keys and `DataLoader`s
- The resolver methods that call on the registered `DataLoader`s to fetch data instead of directly getting it from some datastore

Note that we're still establishing how queries are mapped to resolvers through the `buildWiring` method;
here, the only difference is that the resolvers we reference call `DataLoader`s instead of getting data themselves.


Create an `EntityWiring` class in `src/main/java/resolver/` (new directory).
It should contain the following code:

```java
@Component
public class entityWiring {

  private final DataLoaderRegistry dataLoaderRegistry;

  public entityWiring() {
    this.dataLoaderRegistry = new DataLoaderRegistry();
    this.dataLoaderRegistry.register("singleFieldUnique", newSingleDataLoader());
  }

  @Bean
  public DataLoaderRegistry getDataLoaderRegistry() {
    return dataLoaderRegistry;
  }

  private List<Object> getSingleDataViaBatchHTTPApi(List<String> keys) {
    return keys.stream().map(DummyData::getEntityData).collect(Collectors.toList());
  }
  
  private BatchLoader<String, Object> singleBatchLoader = keys -> {
    return CompletableFuture.supplyAsync(() -> getSingleDataViaBatchHTTPApi(keys));
  };
  
  private DataLoader<String, Object> newSingleDataLoader() {
    return new DataLoader<>(singleBatchLoader);
  }

  public DataFetcher xByIdDataFetcher = environment -> {
    String id = environment.getArgument("id");
    Context ctx = environment.getContext();
    return ctx.singleFieldUniqueDataLoader().load(id);
  };

  public DataFetcher getChildFromParentDataFetcher = environment -> {
    Parent parent = environment.getSource();
    String childId = parent.getChildId().toString();
    Context ctx = environment.getContext();
    return ctx.singleFieldUniqueDataLoader().load(childId);
  };
}
```

As mentioned before, here we are doing a few things:

- Create a `DataLoaderRegistry` and register a new `DataLoader` (created from `newSingleDataLoader()`) to the key `singleFieldUnique`
- Create a data-getting method `getSingleDataViaBatchHTTPApi` that fetches data from our `DummyData` class
- Create a `BatchLoader` and associated `DataLoader` that calls the data-getting method
- Define two resolver methods–– one that will be used to get base-queryable types by `id`, and the other to get `Child` objects when called from expanded `Parent` objects

Now, what is this `Context` that these resolvers are calling?
Well, imagine that our GraphQL service is serving web requests.
Different parties may be making the same, or similar requests, which our `DataLoader`s are fully capable of handling.
However, caching across requests is often not what you want;
instead, we should scope our `DataLoader`s to function and cache on a *per web request* basis.

We do this through the use of a `Context` class, which will hold our `DataLoaderRegistry` and which we will pass to every web request.
Create a `Context` and `ContextProvider` class in `/src/main/java/context/`.
They should contain the following code:

```java
public class Context {

  final DataLoaderRegistry dataLoaderRegistry;

  Context(DataLoaderRegistry dataLoaderRegistry) {
    this.dataLoaderRegistry = dataLoaderRegistry;
  }
  
  public DataLoader<String, Object> singleFieldUniqueDataLoader() {
    return dataLoaderRegistry.getDataLoader("singleFieldUnique");
  }
}
```

```java
@Component
public class ContextProvider {

  final DataLoaderRegistry dataLoaderRegistry;

  @Autowired
  public ContextProvider(DataLoaderRegistry dataLoaderRegistry) {
    this.dataLoaderRegistry = dataLoaderRegistry;
  }

  public Context newContext() {
    return new Context(dataLoaderRegistry);
  }
}
```

Recall that we mapped the `DataLoader` created through `newSingleDataLoader()` to the key `singleFieldUnique` in the `EntityWiring` class.
Here we are allowing that `DataLoader` to be retrieved on a per-request basis through `singleFieldUniqueDataLoader()`, which is what the resolvers we saw before are calling (e.g. `ctx.singleFieldUniqueDataLoader().load(id)`).
If we follow that method back through the registry and the data loader and the batch loader, we eventually arrive at the `getEntityData` method we defined in our `DummyData` class!

Now, we just have two more steps before we're ready to make a request to our GraphQL service.
In our previous GraphQL projects, a `/graphql` endpoint was automatically created for us to make queries to.
However, because we have defined a `Context` that we need to attach to each web request, we need more fine-tuned control of this endpoint.
To do this, we can actually create a Rest Controller that overrides the `/graphql` address. Create a `GraphQLController` class in `/src/main/java/controller/`.
It should contain the following code:

```java
@RestController
public class GraphQLController {

  private final GraphQL graphql;
  private final ObjectMapper objectMapper;
  private final ContextProvider contextProvider;

  @Autowired
  public GraphQLController(GraphQL graphql, ObjectMapper objectMapper,
                           ContextProvider contextProvider) {
    this.graphql = graphql;
    this.objectMapper = objectMapper;
    this.contextProvider = contextProvider;
  }

  @Operation(
      summary = "GraphQL GET request",
      description = "Supply a query and optionally operationName and variables"
  )
  @GetMapping(
      value = "/graphql",
      produces = { MediaType.APPLICATION_JSON_VALUE }
  )
  @CrossOrigin
  public void graphqlGET(
      @RequestParam("query") String query,
      @RequestParam(value = "operationName", required = false) String operationName,
      @RequestParam(value = "variables", required = false) String variables,
      HttpServletResponse httpServletResponse) throws IOException {
    if (query == null) {
      query = "";
    }
    Map<String, Object> variablesMap = new LinkedHashMap<>();
    if (variables != null) {
      variablesMap = objectMapper.readValue(variables, new TypeReference<Map<String, Object>>() {
      });
    }
    executeGraphqlQuery(httpServletResponse, operationName, query, variablesMap);
  }

  @SuppressWarnings("unchecked")
  @Operation(
      summary = "GraphQL POST request",
      description = "Queries from apps like GraphQL Playground go here"
  )
  @PostMapping(
      value = "/graphql",
      produces = { MediaType.APPLICATION_JSON_VALUE },
      consumes = { MediaType.APPLICATION_JSON_VALUE }
  )
  @CrossOrigin
  public void graphql(
      @RequestBody Map<String, Object> body,
      HttpServletRequest httpServletRequest,
      HttpServletResponse httpServletResponse) throws IOException {
    String query = (String) body.get("query");
    if (query == null) {
      query = "";
    }
    String operationName = (String) body.get("operationName");
    Map<String, Object> variables = (Map<String, Object>) body.get("variables");
    if (variables == null) {
      variables = new LinkedHashMap<>();
    }
    executeGraphqlQuery(httpServletResponse, operationName, query, variables);
  }

  private void executeGraphqlQuery(HttpServletResponse httpServletResponse, String operationName,
                                   String query, Map<String, Object> variables) throws IOException {
    Context context = contextProvider.newContext();
    ExecutionInput executionInput = ExecutionInput.newExecutionInput()
        .query(query)
        .variables(variables)
        .operationName(operationName)
        .context(context)
        .build();
    ExecutionResult executionResult = graphql.execute(executionInput);
    handleResponse(httpServletResponse, executionResult);
  }

  private void handleResponse(HttpServletResponse httpServletResponse,
                              ExecutionResult executionResult) throws IOException {
    Map<String, Object> result = executionResult.toSpecification();
    httpServletResponse.setStatus(HttpServletResponse.SC_OK);
    httpServletResponse.setCharacterEncoding("UTF-8");
    httpServletResponse.setContentType("application/json");
    httpServletResponse.setHeader("Access-Control-Allow-Origin", "*");
    String body = objectMapper.writeValueAsString(result);
    PrintWriter writer = httpServletResponse.getWriter();
    writer.write(body);
    writer.close();
  }
}
```

There's a decent amount of code here, but most of it is not relevant to how GraphQL works–– rather, most of this code is just describing how the `/graphql` endpoint should behave, in a fashion not different from how Spring service endpoints are usually described.
Here's a rundown of what this code does:

- First, we're `@Autowire`-ing in the GraphQL service we defined in the `GraphQLProvider` class, along with the `ContextProvider`, which will provide our `Context`s
- Next, we have two methods `graphqlGET` and `graphql` corresponding to the `GET` and `POST` versions of the `/graphql` endpoint, respectively.
There's basically no difference between the two methods–– each receives the same data (albeit in different forms) and calls the fourth method `executeGraphqlQuery`, which builds the execution input with the request context and sends that input to the GraphQL service to resolve
- The last method `handleResponse` just packages whatever response the GraphQL service sends back

And that's it! At least, that's all it takes to set up a GraphQL service that resolves queries quickly and efficiently.
In the next section, we'll take a look at how we can add more complex queries to our current setup.
But, before that, let's try making a query to our service in its current state.

Build our project with `./gradlew build`; you should see this message after a while.

```
...
BUILD SUCCESSFUL in 2s
7 actionable tasks: 7 up-to-date
```

Run our project using `./gradlew bootRun`. If all is well, you should see this message after a few seconds.

```
...
2022-03-15 23:23:21.100  INFO 55237 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-03-15 23:23:21.117  INFO 55237 --- [           main] c.c.SampleApplication   : Started SampleApplication in 3.007 seconds (JVM running for 3.554)
<==========---> 80% EXECUTING [17s]
> :bootRun
```

With [GraphQL-Playground](https://github.com/graphql/graphql-playground), open the `Local URL` option and enter in `http://localhost:8080/graphql`.
Then, in the resulting query page, paste this into the left text box:

```
{
  parentById(id: "92bac69b-ffa1-4cb5-941f-fe1b9c4115da") {
    id
    name
    secondaryId
    childId {
        id
        name
    }
  }
}
```

Once you click the `Run` button, you should see this result:

```
{
  "data": {
    "parentById": {
      "id": "92bac69b-ffa1-4cb5-941f-fe1b9c4115da",
      "name": "parent1",
      "secondaryId": "453b631f-b068-4630-bda4-930571e689c2",
      "childId": {
        "id": "e84a9d26-3350-4c7e-8586-69317262e7e0",
        "name": "child1"
      }
    }
  },
  "extensions": {
    "dataloader": {
    ...
}
```

Notice that now there's an `extensions` dictionary, which shows useful information about the performance of our data loaders.
Also, since we added the base query `childById` we can directly query for `Child` objects.

Now, let's take a look at two more complex queries we can add to our service.
Recall our `schema.graphqls`, where we actually have the `parentBySecondaryId` and `parentByMultiField` base queries that we have not yet implemented.
In our current state, they won't throw errors as long as we don't call them in a GraphQL query.
But, we *would* like to be able to call them, so let's implement them now.

Let's start with `parentBySecondaryId`–– the idea of this query is that we should be able to pass in a parameter that multiple `Parent` objects share and have the service return all of those `Parent`s.
Usually, the primary `id` of an object is unique, which is why the `getById` queries only return one instance of an object;
but, we can imagine cases where filtering by some field returns multiple objects.

In the `DummyData` class, add the following method to get however many `Parent` objects match the given `secondaryId`.

```java
public static List<Object> getEntitiesData(String secondaryId) {
    List<Parent> rawParentData = new ArrayList<Parent>(parentData.values());
    return rawParentData.stream().filter(parent -> parent.getSecondaryId().equals(UUID.fromString(secondaryId))).collect(Collectors.toList());
}
```

Then, create a method in `EntityWiring` to call this data-getter method as a kind of "mock" HTTP/datastore call, along with associated methods for making a `BatchLoader` and a `DataLoader`, as well as a resolver (`Parent`-specific):

```java
private List<Object> getMultipleDataViaBatchHTTPApi(List<String> keys) {
    return keys.stream().map(DummyData::getEntitiesData).collect(Collectors.toList());
}

private BatchLoader<String, Object> multipleBatchLoader = keys -> {
    return CompletableFuture.supplyAsync(() -> getMultipleDataViaBatchHTTPApi(keys));
};

private DataLoader<String, Object> newMultipleDataLoader() {
    return new DataLoader<>(multipleBatchLoader);
}

public DataFetcher xByXMultiDataFetcher = environment -> {
    String secondaryId = environment.getArgument("secondaryId");
    Context ctx = environment.getContext();
    return ctx.singleFieldMultipleDataLoader().load(secondaryId);
};
```

We'll also need to register the new `DataLoader` in the constructor of `EntityWiring`:

```java
public entityWiring() {
    this.dataLoaderRegistry = new DataLoaderRegistry();
    this.dataLoaderRegistry.register("singleFieldUnique", newSingleDataLoader());
    this.dataLoaderRegistry.register("singleFieldMultiple", newMultipleDataLoader());
}
```

Amend the wiring in `GraphQLProvider` accordingly:

```java
private RuntimeWiring buildWiring() {
    return RuntimeWiring.newRuntimeWiring()
        .type(newTypeWiring("Query") // Defines base queries that can be made
            .dataFetcher("parentById", entityWiring.xByIdDataFetcher)
            .dataFetcher("childById", entityWiring.xByIdDataFetcher)
            .dataFetcher("parentBySecondaryId", entityWiring.xByXMultiDataFetcher)
        )
        .type(newTypeWiring("Parent") // Teaches type Parent how to get Child
            .dataFetcher("childId", entityWiring.getChildFromParentDataFetcher)
        )
        .build();
}
```

Finally, in the `Context` class, add a method to return the new data loader from the appropriate key:

```java
public DataLoader<String, Object> singleFieldMultipleDataLoader() {
    return dataLoaderRegistry.getDataLoader("singleFieldMultiple");
}
```

And that's all we need to add for this new query to work.
Let's try it out in GraphQL Playground:

```
{
  parentBySecondaryId(secondaryId: "453b631f-b068-4630-bda4-930571e689c2") {
    id
    name
    secondaryId
    childId {
      id
      name
    }
  }
}
```

The result should look something like:

```
{
  "data": {
    "parentBySecondaryId": [
      {
        "id": "92bac69b-ffa1-4cb5-941f-fe1b9c4115da",
        "name": "parent1",
        "secondaryId": "453b631f-b068-4630-bda4-930571e689c2",
        "childId": {
          "id": "e84a9d26-3350-4c7e-8586-69317262e7e0",
          "name": "child1"
        }
      },
      {
        "id": "9cdc7680-0201-48d5-a53b-3244a9bd4928",
        "name": "parent2",
        "secondaryId": "453b631f-b068-4630-bda4-930571e689c2",
        "childId": {
          "id": "fafe1b57-32bc-43eb-a7b4-41aa05bec16c",
          "name": "child2"
        }
      }
    ]
  },
  "extensions": {
    "dataloader": {
    ...
}
```

As is expected, we are returned two `Parent` objects because we set both of them to have the same `secondaryId`.

Now let's do `parentByMultiField`.
In this query, we can pass any combination of fields defined in the `ParentSearch` input type;
this includes `id`, `name`, `secondaryId`, and `childId`.
Again, this can return multiple objects, if they fit the criteria given.
This kind of query is useful if there is more than just one descriptor for something we want (e.g. get all cars that are red and have three wheels).

The process for adding this query will be the same as before:

- (Since we are not using an actual datastore, we'll need to) define a data-getting method
- Create a mock HTTP call + batch loader + data loader + resolver
- Register the data loader
- Wire the resolver
- Add a method to get the data loader from context

Here is the data-getting method (a bit more complicated this time as we are essentially filtering a list based on values in a map):

```java
public static List<Parent> getEntitiesMulti(LinkedHashMap<String, String> filterCondition) {
    ObjectMapper oMapper = new ObjectMapper();
    // Create predicate from input Map
    Predicate<Map<String, String>> allConditions = filterCondition.entrySet().stream()
        .map(DummyData::getAsPredicate).reduce((entity) -> true, Predicate::and);
    List<Parent> rawParentData = new ArrayList<Parent>(parentData.values());
    List<Map<String, String>> parentDataInMap = new ArrayList<>();
    for (Parent parent : rawParentData) {
        parentDataInMap.add(oMapper.convertValue(parent, Map.class));
    }
    List<Map<String, String>> filteredParentDataInMap =
        parentDataInMap.stream().filter(allConditions).collect(Collectors.toList());
    List<Parent> filteredRawParentData = new ArrayList<>();
    for (Map<String, String> parent : filteredParentDataInMap) {
        filteredRawParentData.add(oMapper.convertValue(parent, Parent.class));
    }
    return filteredRawParentData;
}

// Create dynamic predicate from input Map, checking for equality
private static Predicate<Map<String, String>> getAsPredicate(Map.Entry<String, String> filter) {
    return (Map<String, String> thing) -> thing.get(filter.getKey()).equals(filter.getValue());
}
```

Here are the changes in `EntityWiring`:

```java
public entityWiring() {
    this.dataLoaderRegistry = new DataLoaderRegistry();
    this.dataLoaderRegistry.register("singleFieldUnique", newSingleDataLoader());
    this.dataLoaderRegistry.register("singleFieldMultiple", newMultipleDataLoader());
    this.dataLoaderRegistry.register("multipleFieldMultiple", newMultiFromMultiDataLoader());
}

private List<Object> getMultiFromMultiDataViaBatchHTTPApi(List<LinkedHashMap<String, String>> keys) {
    return keys.stream().map(DummyData::getEntitiesMulti).collect(Collectors.toList());
}

private BatchLoader<LinkedHashMap<String, String>, Object> multiFromMultiBatchLoader = keys -> {
    return CompletableFuture.supplyAsync(() -> getMultiFromMultiDataViaBatchHTTPApi(keys));
};

private DataLoader<LinkedHashMap<String, String>, Object> newMultiFromMultiDataLoader() {
    return new DataLoader<>(multiFromMultiBatchLoader);
}

public DataFetcher xByMultiXMultiDataFetcher = environment -> {
    LinkedHashMap<String, String> criteria = environment.getArgument("where");
    Context ctx = environment.getContext();
    return ctx.multipleFieldMultipleDataLoader().load(criteria);
};
```

In `GraphQLProvider`, we amend the wiring:

```java
private RuntimeWiring buildWiring() {
    return RuntimeWiring.newRuntimeWiring()
        .type(newTypeWiring("Query") // Defines base queries that can be made
            .dataFetcher("parentById", entityWiring.xByIdDataFetcher)
            .dataFetcher("childById", entityWiring.xByIdDataFetcher)
            .dataFetcher("parentBySecondaryId", entityWiring.xByXMultiDataFetcher)
            .dataFetcher("parentByMultiField", entityWiring.xByMultiXMultiDataFetcher)
        )
        .type(newTypeWiring("Parent") // Teaches type Parent how to get Child
            .dataFetcher("childId", entityWiring.getChildFromParentDataFetcher)
        )
        .build();
}
```

And finally, in `Context`:

```java
public DataLoader<LinkedHashMap<String, String>, Object> multipleFieldMultipleDataLoader() {
    return dataLoaderRegistry.getDataLoader("multipleFieldMultiple");
}
```

We should now be able to make a `parentByMultiField` to the GraphQL service.
In GraphQL Playground:

```
{
  parentByMultiField(where: {name: "parent1", secondaryId: "453b631f-b068-4630-bda4-930571e689c2"}) {
    id
    name
    secondaryId
    childId {
      id
      name
    }
  }
}
```

The result should look something like this:

```
{
  "data": {
    "parentByMultiField": [
      {
        "id": "92bac69b-ffa1-4cb5-941f-fe1b9c4115da",
        "name": "parent1",
        "secondaryId": "453b631f-b068-4630-bda4-930571e689c2",
        "childId": {
          "id": "e84a9d26-3350-4c7e-8586-69317262e7e0",
          "name": "child1"
        }
      }
    ]
  },
  "extensions": {
    "dataloader": {
    ...
}
```

We're asking for a `Parent` object with `name = "parent1"` and `secondaryId = "453b631f-b068-4630-bda4-930571e689c2"`, and there's only one of those, which is why only one is returned.
If we removed the `name` field from the query, we would get an identical result to the `parentBySecondaryId` query, as in both queries we end up asking for `Parent` objects with `secondaryId = "453b631f-b068-4630-bda4-930571e689c2"` which there are two of.

That's all for this post!
Today, we set up a GraphQL service in Spring utilizing concepts like `DataLoader`s to ensure our queries are responsive and efficient.
We also looked at implementing more complex queries, like `getByAnyField` and `getByInputType/getByMultipleFields`.
As for where to go from here?

- Try writing [Mutation queries](https://www.graphql-java.com/documentation/execution/#mutations)
- Try implementing your own [scalars](https://www.graphql-java.com/documentation/master/scalars/#writing-your-own-custom-scalars)
