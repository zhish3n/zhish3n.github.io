---
layout: post
title:  "Simple GraphQL Starter"
date:   2022-03-15 18:19:25 +0800
permalink: "/simple-graphql-starter"
description: Creating a lightweight starter Spring service using the GraphQL Java library.
tags: java graphql spring starter
# permalink: /:categories/:year/:month
---

### Creating a Simple Spring Service using GraphQL

In this post I go over setting up a simple Spring service with GraphQL.

First, use [Spring Initializr](https://start.spring.io/) to kickstart setting up a new Spring project.
Use the settings Gradle, Java 8 (I am using [Amazon Corretto 8](https://docs.aws.amazon.com/corretto/latest/corretto-8-ug/downloads-list.html)), and Spring Boot 2.6.4. 
In the `build.gradle` of the new project, add a few essential dependencies:

```
implementation 'com.graphql-java:graphql-java:11.0'
implementation 'com.graphql-java:graphql-java-spring-boot-starter-webmvc:1.0'
implementation 'com.google.guava:guava:26.0-jre'
compileOnly "org.projectlombok:lombok:1.18.16"
annotationProcessor "org.projectlombok:lombok:1.18.16"
```

- The first dependency is the core GraphQL Java library. 
The dependency after that is an addon library that will help us get the GraphQL starter service up and running easier.
- The Guava dependency allows us to create some dummy data objects we will return in our starter service.
- Lombok allows us to add certain annotations to reduce boilerplate code. 

Set up the GraphQL schema in `src/main/resources` called `schema.graphqls`.
It should contain the following code.

```
schema {
    query: Query
}

type Query {
    parent(id: ID): Parent
}

type Parent {
    id: ID
    name: String
    child: Child
}

type Child {
    id: ID
    name: name
}
```

In this schema we are defining one object that our GraphQL service can return–– the `Parent` object, containing an `id`, a `name`, and a `child`.
The `Child` object returned from a `Parent` object contains an `id` and a `name`.
By including the `Parent` object in `type Query`, we are making it queryable through `parent` (more info on this later on).

Next, create a folder called `resolvers` in `src/main/java`.
In it, create two classes called `ParentResolver` and `ChildResolver`.
The classes should look like this (respectively).

```java
@Component
public class ParentResolver {

  private static List<Map<String, String>> parentObjects = Arrays.asList(
      ImmutableMap.of(
          "id", "parent1",
          "name", "PARENT1",
          "childId", "child1"
      ),
      ImmutableMap.of(
          "id", "parent2",
          "name", "PARENT2",
          "childId", "child2"
      )
  );

  public DataFetcher getParentByIdDataFetcher() {
    return dataFetchingEnvironment -> {
      String parentId = dataFetchingEnvironment.getArgument("id");
      return parentObjects
          .stream()
          .filter(parent -> parent.get("id").equals(parentId))
          .findFirst()
          .orElse(null);
    };
  }
}
```

```java
@Component
public class ChildResolver {

  private static List<Map<String, String>> childObjects = Arrays.asList(
      ImmutableMap.of(
          "id", "child1",
          "name", "CHILD1"
      ),
      ImmutableMap.of(
          "id", "child2",
          "name", "CHILD2"
      )
  );

  public DataFetcher getChildByIdDataFetcher() {
    return dataFetchingEnvironment -> {
      Map<String, String> parent = dataFetchingEnvironment.getSource();
      String childId = parent.get("childId");
      return parentObjects
          .stream()
          .filter(child -> child.get("id").equals(childId))
          .findFirst()
          .orElse(null);
    };
  }
}
```

In the first class, we are defining the method that will be called when we make the GraphQL query `parent(id)`.
In either class, we make a dummy list of objects conforming to the model we gave those objects in `schema.graphqls`;
We then have a method `getZZZByIdDataFetcher` that essentially gets whatever `id` we pass in the GraphQL query and tries to resolve it by finding it in the dummy list (hence, resolver).
Note that in `ChildResolver`, since the resolver method is not called from the base query but instead called while we are expanding a `Parent` object, we get the `childId` not through the environment but through the `Parent` object that calls the method.

Next, create a new folder called `service` in `src/main/java`.
Create a Java class named `GraphQLService` in that folder.
The contents of that class should be as follows:

```java
@Service
public class GraphQLService {

  private GraphQL graphQL;

  @Bean
  public GraphQL graphQL() {
    return graphQL;
  }

  @Autowired
  ParentResolver parentResolver;

  @Autowired
  ChildResolver childResolver;

  @PostConstruct
  public void init() throws IOException {
    URL url = Resources.getResource("schema.graphqls");
    String sdl = Resources.toString(url, Charsets.UTF_8);
    GraphQLSchema graphQLSchema = buildSchema(sdl);
    this.graphQL = GraphQL.newGraphQL(graphQLSchema).build();
  }

  private GraphQLSchema buildSchema(String sdl) {
    TypeDefinitionRegistry typeRegistry = new SchemaParser().parse(sdl);
    RuntimeWiring runtimeWiring = buildWiring();
    SchemaGenerator schemaGenerator = new SchemaGenerator();
    return schemaGenerator.makeExecutableSchema(typeRegistry, runtimeWiring);
  }

  private RuntimeWiring buildWiring() {
    return RuntimeWiring.newRuntimeWiring()
        .type(newTypeWiring("Query")
            .dataFetcher("parent", parentResolver.getParentByIdDataFetcher())
        )
        .type(newTypeWiring("Parent")
            .dataFetcher("child", childResolver.getChildByIdDataFetcher())
        )
        .build();
  }

}
```

There are a few things happening here.
First, we're auto-wiring the resolvers we made before into this class.
We're also creating a GraphQL instance that we will use later to execute queries we receive.
In the `init` function (annotated with `@PostConstruct` so it automatically runs) we create a `GraphQLSchema` from the `schema.graphqls` file we created earlier and use it to build our `GraphQL` instance.
Notice that this `init` function calls `buildSchema`, which also calls `buildWiring`.
In `buildWiring`, we are essentially mapping queries defined in `schema.graphqls` to functions we defined in the resolver classes.
`newTypeWiring("Query")` shows base queries that can be run, while `newTypeWiring("Parent")` shows queries that can be run to expand upon a `parent` query (which returns a `Parent` object);
in our case, `Parent` contains a reference to a `Child` object, so we need to define a wiring to let GraphQL know how to get a `Child` from a `Parent`.

And that's it! That's all it takes to set up a super simple and lightweight Spring service running GraphQL.
Build our project with `./gradlew build`; you should see this message after a while.

```
...
BUILD SUCCESSFUL in 2s
7 actionable tasks: 7 up-to-date
```

Then, run our project using `./gradlew bootRun`. If all is well, you should see this message after a few seconds.

```
...
2022-03-15 23:23:21.100  INFO 55237 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-03-15 23:23:21.117  INFO 55237 --- [           main] c.c.udl.discovery.DiscoveryApplication   : Started DiscoveryApplication in 3.007 seconds (JVM running for 3.554)
<==========---> 80% EXECUTING [17s]
> :bootRun
```

It will continue to stay at `80% EXECUTING`, which indicates that the Spring server is up and running.
To close the server, press `control + c`. 
GraphQL automatically creates an endpoint `/graphql` for us to query although, unfortunately, there is no easily accessible UI (like Swagger) for us to look at.
Instead, we need to get something like [GraphQL-Playground](https://github.com/graphql/graphql-playground) to help us visualize our queries.
With GraphQL-Playground open, hit the `Local URL` option and enter in `http://localhost:8080/graphql`.
Then, in the resulting query page, paste this into the left text box:

```
{
  parent(id: "parent1") {
    id
    name
    child
  }
}
```

Once you click the `Run` button, you should see this result:

```
{
  "data": {
    "parent": {
      "id": "parent",
      "name": "PARENT1",
      "child": {
        "id": "child1",
        "name": "CHILD1"
      }
    }
  }
}
```

You can edit the query to get `parent2` instead, or add/remove whichever fields you want GraphQL to return.

That just about wraps up this post!
Again, this is a super lightweight starter service showing only basic functionalities of GraphQL.
There are many other things GraphQL can do and other ways of implementing them in Spring that I'll explore in other posts.
As for where to go from here?

- Add other types (with more fields) and resolvers.
- Experiment with queries that return lists (e.g. `parent(name: String): [Parent]`)
- Check out my other GraphQL posts that dive into more complex queries and other useful GraphQL libraries!