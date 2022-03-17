---
layout: post
title:  "GraphQL Java and Spring with Databases"
date:   2022-03-18 18:19:25 +0800
permalink: "/graphql-java-spring-database"
description: A tutorial on building a GraphQL Java service in Spring with an H2 database.
tags: java graphql spring h2
# permalink: /:categories/:year/:month
---

### Creating a GraphQL Java service using Spring and H2

In this post I go over setting up a Spring service with GraphQL and H2.
This post should be read as a continuation of the [simple GraphQL starter post](./simple-graphql-starter), as that post goes over some basic expectations of GraphQL and in this post we'll be building off some concepts we explored in the other post.

First, use [Spring Initializr](https://start.spring.io/) to kickstart setting up a new Spring Boot project.
Use the settings Gradle, Java 8 (I am using [Amazon Corretto 8](https://docs.aws.amazon.com/corretto/latest/corretto-8-ug/downloads-list.html)), and Spring Boot 2.6.4.
In the `build.gradle` of the new project, add these dependencies:

```
implementation 'com.h2database:h2:2.1.210'
implementation 'org.springframework.boot:spring-boot-starter-data-jpa:2.6.4'
implementation 'com.graphql-java:graphql-spring-boot-starter:5.0.2'
implementation 'com.graphql-java:graphql-java-tools:5.2.4'
compileOnly "org.projectlombok:lombok:1.18.16"
annotationProcessor "org.projectlombok:lombok:1.18.16"
```

- The first dependency is the database system we will embed in our application that will store the data we want the GraphQL service to be able to fetch.
- The second dependency provides repository support, allowing us to query from the H2 database.
- The third and fourth dependencies are add-ons to the base GraphQL Java library that introduce some cool features.
- Lombok allows us to add certain annotations to reduce boilerplate code.

Set up the GraphQL schema in `src/main/resources/graphql/` called `schema.graphqls`.
It should contain the following code.

```
type Query {
    parent(id: ID!): Parent
}

type Parent {
    id: String!
    name: String!
    child: Child
}

type Child {
    id: String!
    name: String!
}
```

In this schema we are defining one object that our GraphQL service can return–– the `Parent` object, containing an `id`, a `name`, and a `Child`.
By including the `Parent` object in `type Query`, we are making it queryable through `parent(id)`;
the `Child` object is not exposed here so we will only be able to get `Child` data from related `Parent` objects.
Notice the exclamation marks `!` used in the schema;
these indicate where a field is required–– so in our case, a `Parent` must have an `id` and `name`.

Next, create a folder called `resolver` in `src/main/java/`.
In it, create two classes called `ParentQueryResolver` and `ParentResolver`.
They should look like this:

```java
@Component
public class ParentQueryResolver implements GraphQLQueryResolver {

    private ParentRepository parentRepository;

    ParentQueryResolver(ParentRepository parentRepository) {
        this.parentRepository = parentRepository;
    }

    public Optional<Parent> parent(String id) {
        return parentRepository.findById(id);
    }

}
```

```java
@Component
public class ParentResolver implements GraphQLResolver<Parent> {

    private ChildRepository childRepository;

    ParentResolver(ChildRepository childRepository) {
        this.childRepository = childRepository;
    }

    public Optional<Child> child(Parent parent) {
        return childRepository.findById(parent.getChildId());
    }
}
```

Unlike in the last post, here we do not have the dummy `Map<String, String>` datastores.
Instead, we have a `ParentRepository` and a `ChildRepository` (we will define these in the next step) which we instantiate in the class constructors.
Our "data-fetching" methods have also changed.
In the `ParentQueryResolver`, we take the `id` as an input and return the result of calling `findById` with that `id` on the repository.
In the `ParentResolver`, we take a `Parent` as an input and return the result of calling `findById` with on the `ChildRepository` with that `Parent`'s `childId`.
You may observe that these methods are analogous to the `DataFetcher` methods from the other post, and you would be correct;
we will touch on this similarity in a later section.

Create a new folder called `repository` in `src/main/java/`.
Create two Java interfaces named `ParentRepository` and `ChildRepository` in that folder.
The (very bare) contents of those interfaces should be as follows:

```java
public interface ParentRepository extends JpaRepository<Parent, String> {
}
```

```java
public interface ChildRepository extends JpaRepository<Child, String> {
}
```

Also create a new folder called `model` in `src/main/java/` and two files in that folder called `Parent` and `Child`.
The contents of those classes should be:

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Parent {

  @Id
  private String id;
  private String name;
  private String childId;

}
```

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Child {

    @Id
    private String id;
    private String name;

}
```

These classes define the objects that will be returned by the Jpa repositories.
They are annotated in such a way that upon starting the Spring service H2 will automatically create tables capable of holding objects with these definitions.

Create a file called `data.sql` in `src/main/resources/`:

```
insert into parent (id, name) values ('f563e845-da10-46e7-875e-340595a07ecc', 'Test1', 'd668de84-554b-4017-853b-d089fb644542');
insert into parent (id, name) values ('4872a3b8-1f53-4f80-b496-7443bfe8eb17', 'Test2', 'ee40272a-8a05-4ee3-b2f1-2579ba93b83f');
insert into child (id, name) values ('d668de84-554b-4017-853b-d089fb644542', 'Test3');
insert into child (id, name) values ('ee40272a-8a05-4ee3-b2f1-2579ba93b83f', 'Test4');
```

After all relevant tables have been automatically created by H2, `data.sql` is run to populate those tables.
Here we are just inserting two `Parent` rows and two `Child` rows (each belonging to one parent).
Lastly, we'll need to configure our `src/main/resources/applicaton.yml` file to properly run our H2 database:

```
spring:
  application:
    name: entity_project
  datasource:
    url: jdbc:h2:mem:testdb
    driverClassName: org.h2.Driver
    username: username
    password: password
    hikari:
      connection-timeout: 2000
      initialization-fail-timeout: 0
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    defer-datasource-initialization: true
    properties:
      hibernate:
        show_sql: true
        format_sql: true
        enable_lazy_load_no_trans: true
```

And that's it! That's all it takes to set up our Spring service with GraphQL Java and H2.
You might notice that, unlike in the project we built in the last post, we don't have a `service` class here.
Additionally, we don't have the `buildWiring()` method where we're wiring together our `resolver` methods with GraphQL query commands.
That is because [GraphQL Java Tools](https://github.com/graphql-java-kickstart/graphql-java-tools) does this for us automatically.
Recall the query resolver classes we wrote earlier, and how they were sort-of analogous to the `DataFetcher`-returning resolvers from the other project.
Here, a class implementing `GraphQLQueryResolver` is treated as defining some method included in `type Query`.
GraphQL Java Tools looks for a method with the same name and taking the same input (in our case, `parent`) to "wire" with the GraphQL query command.
A class implementing `GraphQLResolver<Type>` becomes a resolver for sub-queries/type-expansions within that `Type`;
in our case, our `GraphQLResolver<Parent>` deals with resolving expanding the `Child` reference within `Parent` objects.
This is automatically wired as well, as we are providing the parent class and a name-input-matching resolver returning the target child class.

On top of that, [GraphQL Spring Boot Starter](https://github.com/graphql-java-kickstart/graphql-spring-boot) automatically fetches any `graphqls` schema files in `resources`.
For a larger project, we could split up our `schema.graphqls` into multiple smaller schema files and it would automatically compile them all.
Combined with GraphQL Java Tools, this means we don't have to do any of the schema building/compilation or resolver wiring–– it all just "magically" works! 

Let's now try building our project with `./gradlew build`; you should see this message after a while.

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

Like in the previous project, GraphQL automatically creates an endpoint `/graphql` for us to query.
Again, we'll need an app like [GraphQL-Playground](https://github.com/graphql/graphql-playground) to help us visualize our queries.
With GraphQL-Playground open, hit the `Local URL` option and enter in `http://localhost:8080/graphql`.
Then, in the resulting query page, paste this into the left text box:

```
{
  parent(id: "f563e845-da10-46e7-875e-340595a07ecc") {
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
      "id": "f563e845-da10-46e7-875e-340595a07ecc",
      "name": "Test1",
      "child": {
        "id": "d668de84-554b-4017-853b-d089fb644542",
        "name": "Test3"
      }
    }
  }
}
```

You can edit the query to get the other `Parent` (`Test2`) instead, or add/remove whichever fields you want GraphQL to return.

And that's it!
In this post, we covered setting up a Spring service using GraphQL and an H2 datastore.
We used two GraphQL Java addons to help vastly simplify the process of wiring different parts of the service together.
Although we explored a different way of setting up a GraphQL service, we've yet to take a look at how we can implement more complex GraphQL queries, so let's do that in the next post!
