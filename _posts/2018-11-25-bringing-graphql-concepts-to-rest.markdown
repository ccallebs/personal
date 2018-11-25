---
title: Bringing GraphQL Concepts to REST
layout: post
---

With GraphQL in vogue and traditional REST apis feeling like antiquated technology, I started thinking of ways REST could catch up to the tooling, convenience, and rapid prototyping offered by GraphQL and its associated frameworks. I'm familiar with [Introspected REST](https://introspected.rest), but find that it fundamentally changes the way most folks interact with a REST API. I believe we can develop a solution that allows us to keep the workflow but eliminate the rigidity. First, I'd like to describe what I see as the major strengths of GraphQL with the intention of bringing them to REST framework.

## Strong Typing Support

The GraphQL type system provides a simple way to ensure data integrity and offers context to the shape of your return objects. We want our ideal REST framework to warn us when we're passing a string to an integer field. We want "compile" errors when we try to return a User object for a Comment attribute. And ideally we would like the power to define [union types](https://graphql.org/learn/schema/#union-types) to keep our data flexible yet typesafe.

## Easy Schema Exploration

From my admittedly limited experience with GraphQL, comprehensive schema exploration is one of the more compelling reasons to adopt. Using tools like [GraphiQL](https://github.com/graphql/graphiql), you can browse the shape of an application's data without knowing _anything_ about it beforehand. I can't overstate how important this is. API documentation quality varies as much as the code quality underneath. You can use tools like [Swagger](https://swagger.io/), but at the cost of slowing down development.

This seems like the easiest problem to address as long as we explicitly define the shape of our data in the API creation process.

## Dynamic Return Data
One of the major boons to GraphQL is the ability to treat your queries like a buffet line, choosing which attributes you’d like from each model while ignoring the unimportant details. This comes at a cost for GraphQL -- all API requests must be made using the POST method and HTTP caching becomes impossible as a result.

I believe dynamic return data would be possible with REST if we slightly modified the way we think of an API. Instead of static resources, we could provide the building blocks necessary to create ad-hoc queries at will. Unlike GraphQL, it would require two requests to compose the initial query. But also unlike GraphQL, the endpoints would be cacheable and the developer experience would be nearly as good.

Let’s pretend we’ve built a framework with a `/shapes` endpoint that accepts POST requests. We could create a named shape at will:

<pre>
{
  'shape_key': 'user_with_comments',
  'model': 'User',
  'fields': ['id', 'name', 'email'],
  'includes': [
    { 'model': 'Comments', 'fields': ['text', 'rep_score'] }
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
        { 'text': 'Great data shape!', score: 1000 },
      ]
    }
  ]
}
</pre>

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

I have a rather selfish interest in building a framework offering the above. I love Ruby. I don't think there's a better language for programmer happiness. It's a joy to write and _usually_ a joy to read. RSpec/MiniTest are fantastic and are rivaled by very few competitors (Elixir/Clojure come to mind). However, I can see the writing on the wall. JavaScript has a vibrant tooling/framework ecosystem while Ruby's maturity has slowed down the number of "big swings". We're content and complacent as Ruby developers. Ruby (and Rails) was at one time the best language to "get shit done." I'm not sure if that's true anymore.

I think something like the above could be a **major** boon to Ruby adoption and could remind folks of the kind of things Ruby is capable of on the server. So with that said, I'll be trying to build it!
