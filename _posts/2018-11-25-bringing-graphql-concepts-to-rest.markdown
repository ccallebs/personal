---
title: Bringing GraphQL Concepts to REST
layout: post
---

<em>Last updated: 11/29/2018</em>

Well, it's 2018 and GraphQL has solidifed itself as the premiere method handling client-side data. When looking at the tooling, convenience, and rapid protyping that Facebook's query language promotes, it's easy to assume that traditional REST APIs are headed the way of the condor. I mean, check out [this StackOverflow question](https://stackoverflow.com/questions/15056878/rest-vs-json-rpc) that originally asked for advice for choosing between REST and JSON-RPC:

> Update 2018: GraphQL offer several advantages over traditional options, specifically it obviates the need to manually normalize, denormalize and cache data between server and client, also it aggregates API calls from UI components without creating a singleton client state which allows developers to build UI components independently and in parallel.

REST is dead, right?

I hope not. REST is a really straightforward way of interacting with a webserver without adding any additional abstractions. It would be a shame if we stopped innovating around it because a new, cooler kid moved to school. With that in mind, I started thinking of the ways REST could try and catch up to the features offered by GraphQL and it's associated ecosystem. Along the way, and I encountered [Introspected REST](https://introspected.rest) and agreed with the issues that it sets out to solve. However, Introspected REST doesn't scratch my particular itch, for a couple of reasons:
- I think our endpoints should be predictable and we shouldn't need to query metadata to figure out endpoint behavior.
- I don't think GraphQL's resolvers are the main thing we need to bring to REST, which is what I understand Introspected REST is trying to do.

I'm sure we can develop a framework that allows us to keep the REST paradigm we're used to but eliminate the rigidity. There are a number of great aspects of GraphQL that I think we can bring along with small modifications and very little engineering overhead. Let's explore a few.

## Strong Type Support

The GraphQL type system provides a simple way to ensure data integrity and offers context to the shape of your return objects. We want our ideal REST framework to warn us when we're passing a string to an integer field. We want "compile" errors when we try to return a User object for a Comment attribute. And ideally we would like the power to define [union types](https://graphql.org/learn/schema/#union-types) to keep our data flexible yet typesafe. Above all, we want the schema to be easy to define and not get in the way any more than necessary. Let's give it a shot:

<pre>
class User
  include Schema::Model

  field :id, type: :integer
  field :name, type: :string
  field :email, type: :string
  field :is_admin, type: :boolean

  has_many :comments, class: Comment
end

class Comment
  include Schema::Model

  field :text, type: :string

  belongs_to :author, class: User, foreign_key :user_id
end
</pre>

As you can see, I'm simply picturing a more explicit ActiveRecord. With the added type information, we can provide schema data to consumers while also generating the associated migrations automatically.

## Easy Schema Exploration

From my admittedly limited experience with GraphQL, comprehensive schema exploration is one of the more compelling reasons to adopt. Using tools like [GraphiQL](https://github.com/graphql/graphiql), you can browse the shape of an application's data without knowing _anything_ about it beforehand. I can't overstate how important this is. API documentation quality varies as much as the code quality underneath. You can use tools like [Swagger](https://swagger.io/), but at the cost of slowing down development.

This seems like the easiest problem to address as long as we explicitly define the shape of our data in the API creation process. Using the models defined above, we could build our framework to have a `/schema` endpoint that accepted GET requests. Depending on the content-type, we could return a generated webpage with the associated schema information or a JSON representation of the same.

It would even be trivial to add permissions-based documentation based on the user currently logged in!

## Dynamic Return Data
Another one of the major advantages of GraphQL is the ability to treat your queries like a buffet line, choosing which attributes you’d like from each model while ignoring the unimportant details. This comes at a cost for GraphQL -- all API requests must be made using the POST method and HTTP caching becomes impossible as a result.

I believe dynamic return data would be possible with REST if we slightly modified the way we think of an API. Instead of static resources, we could provide the building blocks necessary to create ad-hoc queries at will. Unlike GraphQL, it would require two requests to compose the initial query. But also unlike GraphQL, the endpoints would be cacheable and the developer experience would be nearly as good.

Let’s pretend our framework also has a `/shapes` endpoint accepts POST requests. We could create a named shape at will:

<pre>
{
  'key': 'user_with_comments',
  'model': 'User',
  'fields': ['id', 'name', 'email'],
  'includes': [
    { 'model': 'Comments', 'fields': ['text', 'score'] }
  ]
}
</pre>
Now, we could access this data at the `/shapes/user_with_comments` endpoint. If we were to perform a standard GET request, it could return the following:

<pre>
{
  [
    {
      'id': 1,
      'name': 'Chuck Callebs',
      'email': 'chuck@callebs.io',
      'comments': [
        { 'text': 'Great data shape!', 'score': 1000 },
      ]
    }
  ]
}
</pre>
So, what does this buy us? A few things, actually:

- We're only returning the data that we need.
- We can now **cache the data at the request layer**, something GraphQL cannot do.
- Because we're storing the shape of this endpoint, we can automatically add it to our API documentation at `/schema`.

## Automatic Schema Updates
In the [Prisma](https://www.prisma.io/) framework, an engineer only has to modify their schema and the server will automatically update the data store. The migrations will run and unless there’s a destructive action, no further participation is necessary by the engineer. For destructive actions, the participation is only slightly [more involved](https://www.prisma.io/docs/data-model-and-migrations/migrations-asdf/). Here's an example of a column rename:

<pre>
type Story {
  content: String @rename(oldName: "text")
}
</pre>
Then after the deploy, it's up to the engineer to remove the `@rename` annotation manually.

I'm on the fence with how I feel about this. On one hand, this is a very slick way of handling simple migrations/schema changes. On the other, this approach drastically lowers the barrier for accidental data loss. I think some sort of middle ground exists. We could create a framework that automatically generated the migrations, but would require an affirmative nod from the engineer that everything was good. This middle ground also eliminates some of the "magic" associated with the Prisma approach.

## Why don't you just use GraphQL?

Good question.

Plainly put, I don't really think it's necessary. I think what folks needed to do all along was explcitly design their schema and build tooling around it. I think this would be fairly easy to do in Ruby, and it sounds like good challenge and side project. So, who wants to help me build it?
