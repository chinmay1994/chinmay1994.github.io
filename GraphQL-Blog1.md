# Introduction

We at Deepintent have been GraphQL over the past 3 years for our platform. We follow a microservice architecture, which uses [GraphQL Java](https://www.graphql-java.com/) for the various microservices, and a Apollo GraphQL service which stitches the schemas from all the micro-services together to create one executable schema (and more importantly, a single endpoint) for our front-end applications to query. 

This has great benefits for the front-end web applications, as those apps can query only for the required fields, which reduces payload and results in faster loading times compared to REST, which ultimately makes our users happy 👏 

However, using GraphQL Java is not that rosy in the backend, where certain constraints result into tricky, sometimes downright frustrating situations! This article explains one such problem that we ran into, and a possible solution to overcome it.

## Premise

At Deepintent, as mentioned earlier, we use GraphQL java in the backend. Our projects use spring-boot, and the [graphql java kickstart](https://github.com/graphql-java-kickstart/graphql-spring-boot) library that helps us to avoid boilerplate code, like creating a graphql server, mapping graphql schema to Java classes and so on. 

We describe our graphql schema in static schema (.graphqls) files, which are located in the classpath. Each microservice defines its schema, and this schema is merged into one common schema at a gateway service, which is built using Apollo graphql.

By using Graphql java kickstart, we can map the types defined in graphql to Java classes. However, the benefit of graphql is returning only the data that has been asked for, and thus, the Java representation of a type may not alwyas be fully present in one class. In graphql-java-kickstart, there are empty interfaces called `GraphQLResolver`, which allow you to define resolvers that may not be directly map to the type.

For example, consider the following graphql schema:
```
...
type Person {
    id: ID!
    name: String
    address: Address
}

type Address {
    street: String!
    zipcode: String!
    landmark: String
}

type Query {
    people: [Person]
}
...
```
In this scenario, the Java representation of `type Person` is divided into two classes.
```
public class Person {
    private Long id;
    private String name;
    //Constructors, getters and setters 
}

public class PersonResolver implements GraphQLResolver<Person> {
    ..
    public Address address(Person person) {
        return personDataLoader.load(person.getId());
    }
    ..
}
```

Those familiar with graphql can see that the use of `Dataloader`, which is a library that helps you to batch results for a query to resolve additional fields. This typically reduces the number of calls that you make to your database, thus eliminating the n+1 problem. This article assumes you understand how dataloaders work.

Dataloaders work asynchrnously, and they are invoked by graphql execution on every "tick". Thus, using them outside the context of your query execution is not trivial, there is a dataloader registry present in your execution, and your dataloader should be registered, and you need to sacrifice a young lamb to the graphql gods before it works. But I digress.. This is not a rant about dataloaders, and for further explanation of this, read the solution paragraph, which explains this in more detail.

## The Problem
Our application is a standard spring application, which follows the service-repository architecture. It is fairly common, repositroy classes manage access to database, services have business logic. In REST, you typically have a controller layer, which handles requests. In GraphQL, that is replaced by Query/Mutation resolvers, which serve as the entry-point for your query/mutation.

Now, the fun begins. You have a query resolver for the `people` query, which will return a List of type `Person`. However, the definition of Person is in two classes, and the PersonResolver class cannot be called without the people query, as there is no other entry point for its execution. 

In this scenario, consider that you have to fetch addresses of some person types, and send it to some other service. How will you fetch this list of type Person? 

(Hint: Think about what you just wrote in the schema above..)

That's right! You have already done the hardwork, that fetches an address for all the people and wrapped it into a graphql query called `people`. Time to celebrate ?? Not quite.

So now if you see, there is no way to fetch the address for a person, without calling the `people` query. And that query is hosted in your own server. How do you call itself? In our architecture, we have a gateway that holds the final schema. So we can call our gateway to access data in our server, from the same server! Dosen't this make you want to bang your head on that wall in front of you? For me, it sure did!

# The solution
After working very hard for a day to find a solution (read as: searched on stackoverflow for an answer for 2 hours, and crying on not getting anywhere for 6 hours), we decided to take the matter in our own hands. That is when we stumbled across [this](https://github.com/graphql-java/graphql-java/blob/master/src/main/java/graphql/GraphQL.java) class in the graphql-java library. Please take a moment to read the beautiful comment on the top of the class, which explains the exact purpose of the class. For the scope of this article, I'll give a TL;DR version. It lets you call your resolvers from inside other resolvers.

As per the documentation, creating this class per request from the schema provided to it is a cheap operation, as compared to generating the entire schema. As per our app structure, we have a `AuthContext` object, which as the name suggests, contains authorisation and other contextual information and is set in the webserver on every request. We added this GraphQL class mentioned above to this context object, and it automatically became available to every mutation/query from the datafetching environment.

From there on, it is a simple matter of calling the `people` query like this:
```
{
    people {
        id
        address {
            street
            zipcode
        }
    }
}
```
This query is called via the graphql object like this:
```
ExecutionInput executionInput = ExecutionInput.newExecutionInput()
            .query(query)
            .context(env.getContext())
            .dataLoaderRegistry(env.getExecutionContext().getDataLoaderRegistry())
            .root(env.getRoot()).build();
    ExecutionResult execute = graphQL.execute(executionInput);
```
Where query is the people query, and the dataloader registry is passed so that the the dataloader in the PersonResolver class is triggered. If your dataloader is not `registered` at the time of query execution, the address field is never resolved, and the query appears "hanged", or runs infinitely (learned this the hard way) :/.

So, after you put all of this together, you can call queries/mutations which are defined in your local schema. yay!

# Conclusion
???? 
