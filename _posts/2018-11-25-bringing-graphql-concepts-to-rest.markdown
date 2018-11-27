---
title: Bringing GraphQL Concepts to REST
layout: post
---

<em>Last updated: 11/26/2018</em>

With GraphQL in vogue and traditional REST APIs feeling like antiquated technology, I started thinking of ways REST could catch up to the tooling, convenience, and rapid prototyping offered by GraphQL and its associated frameworks. I appreciate [Introspected REST](https://introspected.rest) and the issues it sets out to solve, but I chose to explore a different path for a couple of reasons:
- I think our endpoints should be predictable and we shouldn't need to query metadata to figure out endpoint behavior.
- I don't think we should bring resolvers to REST, which is what I understand Introspected REST is trying to do.

I believe we can develop a framework that allows us to keep a standard REST paradigm but eliminate the rigidity. There are a number of great aspects of GraphQL that I think we can bring along with small modifications and very little engineering overhead.

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

## Dynamic Return Data
One of the major advantages of GraphQL is the ability to treat your queries like a buffet line, choosing which attributes you’d like from each model while ignoring the unimportant details. This comes at a cost for GraphQL -- all API requests must be made using the POST method and HTTP caching becomes impossible as a result.

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
Additionally, this shape would automatically be added to the generated `/schema` information.

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

I have a rather selfish interest in building a framework offering the above. Simply put, I love Ruby and I don't think there's a better language for programmer happiness. It's a joy to write and _usually_ a joy to read. The testing ecosystem has only a handful of rivals (Elixir/Clojure come to mind). However, I can see the writing on the wall. JavaScript has a vibrant tooling/framework ecosystem. We're content and complacent as Ruby developers. Our tools work great, which has prevented us (or me, at least) from continually asking what could be better. Although you can still get a _ton_ done with Ruby (and specifically Rails), I'm not sure we're the fastest gun in the west anymore.

I think something to comparable to Prisma / AppSync would be **major** boon to Ruby adoption. It could remind folks about the kind of things Ruby is capable of on the server. Who wants to help me build it?
