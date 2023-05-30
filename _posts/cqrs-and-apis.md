---
tags:
  - CQRS
  - Api
  - REST
  - GraphQL
  - Just Thoughts
---

# CQRS and APIs
In every web service, you probably have an API - often it is some kind of REST or HTTP api. When applying CQRS to your application, you might struggle a bit, choosing the design for the API. Should you do REST? Or what about the GraphQL? Or gRPC? There are so many options, and many of these can solve you need.

# Just a brief recap:

## CQRS (Command Query Responsibility Segregation)
Many have seen a very complicated diagram with command handlers, domain models and an Event Store. This might overcomplicates CQRS a bit - at least seen from the client. Basically one of the first things you learn when programming is CQRS: create methods for getting and methods for setting state. Your api will be seperated in Commands (like `RenameUser(42, "Mickey")`) and Queries (like `GetUser(42)`). Both are probably some kind DTO to allow passing data over the wire.
Often CQRS is paired with Event Sourcing, which makes it a bit more complicated to read you writes, since your read models are eventual consistent. In that case, the command can return some kind of cursor which you can pass along to the query. The query can the wait for it's read models to catch up with that cursor, before returning data to the client.

## REST
There are many aspects of REST. For this discussion, the relevant part is that your data is seen as resources which you can Create, Read, Update and Delete (CRUD), often though the HTTP protocol.

## GraphQL
GraphQL is another way of designing your APIs. On the basic level, it allows you performs queries and execute mutations. Mutations are defined as a command followed by a query. On top on that is allows you to specify which fields and which relations you want to load (for both queries and mutations). Just to give a quick example of queries and mutations:
```graphql
mutation {
  updateUser(id: 42, firstName: "Mickey") {
    user {
      fullname
    }
  }
}

query {
  user(id: 42) {
    fullname
  }
}
```

## RPC
I will not go into details about RPC - but you can easily use gRPC, SignalR or similar. Basically you just add a layer handling the transport and you will have access to methods on the server. A drawback is that you need an implementation for the specific protocol for your client. Luckily all RPC libraries have that for the major languages.


# Choosing the right API
So now you have choosing CQRS and you need to expose an API. What do you do?

## Roll your own
You could roll you own API - and I actually think it is an okay approach. It should be fairly trivial to add two HTTP endpoints for commands and queries respectively:

```
PUT /execute/{commandName} 
GET /query/{queryName}
```

It aligns perfectly with CQRS. For typesafety you should then generate some typings or packages to share between server and client - that's a bit more complex if you want to have different languages on the server and client.

Futher you should decide if GET is right verb for you queries, as it does not allow a body. You could go with POST or PUT for simplicity - or can be fancy with custom HTTP verbs:

```
EXECUTE /{commandName} 
QUERY /{queryName}
```

In the end, you will design your own RPC protocol, which might fit your needs better than existing RPC libraries - just be aware that you probably can find something already built.

## Using REST
Often people comes from a world of REST apis, and a natural choice would be what you already know and love. Choosing CQRS tends to come from the acknowledgement that your domain is (or you expect it to be) more complex than CRUD. If that is the case, it seems counter intuitive to choose a CRUD based API for your system.
Another argument for not choosing REST, is the fact that commands might not belong to a specific resource - or might belongs to multiple resources. So how do you name your route?
And last, why complicate naming your REST routes and choosing the right verb, when you already have a name for your command, that perfectly describe what is the intent.
One thing REST could be used for, is viewing your queries and command as resources - i.e.:
```
GET /commands
GET /commands/{commandName}
GET /queries
...
```
That way you can expose the capabilities for you system. Executing a command, should not be a POST or PUT since we cannot create or modify commands from the client.

REST seems more useful for exploring the API and for interacting with the system. I would **not** recommend using REST for you CQRS systems.


## GraphQL
GraphQL can be good fit for CQRS, since it already have the separation of mutations and queries.

The query part aligns with a idea of queries in CQRS, and it also give you the possibility of deeper queries and only selecting the fields you need. One thing though is the owner of the query. With GraphQL the client designs the query, and that might be inefficient on the server. With CQRS you can have a very specific and complex query that is calculated before it is needed and thus is very efficient at query time.

For mutations there is a bit more to it. Since mutations are a composition of a command and a query it does not immediate feel like a fit. You can have your mutations return a `status: "OK"` and let the client do the query afterwards, but that seems to defeat the purpose of GraphQL. Instead you can choose to look at your application as CQRS - until the API layer. Allowing the API to execute a command and afterwards to a query, hides away some complexity for the user. Especially when also having eventual consistency. A mutation will then

- Execute a command and get back a cursor
- Wait for whatever queries the client have requested to catch up
- Execute the queries

If the client haven't requested anything, the mutation returns as fast as the command can execute. Otherwise it will hang for a while, but to the client it still saves another round trip to the api (as the client probably want to do a query to read the write).

GraphQL is still a relative new player and there are not a lot of libraries out there (compared to REST). It is a bit more complex than REST or plain HTTP. But if you spend some time to learn it, it can be a good fit for CQRS.