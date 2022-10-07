---
tags:
  - NestJS
  - GraphQL
---

## Three years of GraphQL and NestJS

For the last three years I have been working a lot with GraphQL in NestJS ([@nestjs/graphql](https://www.npmjs.com/package/@nestjs/graphql)), and in general I really think this is a great framework. Since NestJS is opinionated, it should be easy to figure out the right way to organize your GrahpQL resolvers. But it isn't and it bugs me. There is some guidance of how to do it, but it seems to be some simple use cases. In a real world application that have just a minimum of complexity, these guides falls short.

In the following, I will explore this schema:

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Comment {
  author: User!
  post: Post!
}

type Post {
  id: ID!
  auther: User!
  comments: [Comment!]!
}

type Query {
  user(id: ID!): Author!
  post(id: ID!): Post!
}


type Mutation {
  registerUser: User!
  createPost: Post!
  addComment(postId: ID!): Comment!
}
```

Even though this is simple example, it is very likely that it will (at least at some point) be split into two modules:
```
- users (User)
- posts (Post, Comments)
```

### Guide me!
Following the [guide](https://docs.nestjs.com/graphql/resolvers) these resolvers would be created:

```ts
// users/users.resolver.ts
@Resolver(User)
export class UsersResolver {
  @Query(() => User)
  user(@Args('id', { type: () => ID }) id: string) {}

  @Mutation(() => User)
  registerUser() {}

  @ResolveField(() => String)
  name(@Parent() user: User) {}

  @ResolveField(() => [Post])
  posts(@Parent() user: User) {}
}

// posts/posts.resolver.ts
@Resolver(Post)
export class PostsResolver {
  @Query(() => Post)
  post(@Args('id', { type: () => ID }) id: string) {}

  @Mutation(() => Post)
  createPost() {}

  @Mutation(() => Post)
  addComment(@Args('postId', { type: () => ID }) postId: string) {}

  @ResolverField(() => User)
  author(@Parent() post: Post) {}

  @ResolverField(() => [Comment])
  comments(@Parent() post: Post) {}
}

// posts/comments.resolver.ts
@Resolver(Comment)
export class CommentsResolver {
  @ResolverField(() => User)
  author(@Parent() comment: Comment) {}

  @ResolverField(() => Post)
  post(@Parent() comment: Comment) {}
}
```

This seems fine by at first glance, but it comes with some problems.


### One resolver, many responsibilities
First thing to notice, is that the resolvers have quite a lot going on: it uses @Query, @Mutation and @ResolveField. And more critical: why is the queries and mutation put in the PostsResolver and not the CommentsResolver - probably because it is in the posts module, but there is no guarentee that a module have a main entity with the same name as the module. A better separation would be to split resolvers into query-, mutation- and entity resolvers. For the comments module it looks like:

```typescript
// posts/query.resolver.ts
@Resolver()
export class QueryResolver {
  @Query(() => Post)
  post(@Args('id', { type: () => ID }) id: string) {}
}

// posts/mutation.resolver.ts
@Resolver()
export class MutationResolver {
  @Mutation(() => Post)
  createPost() {}

  @Mutation(() => Post)
  addComment(@Args('postId', { type: () => ID }) postId: string) {}
}

// posts/post.resolver.ts
@Resolver(Post)
export class PostResolver { // <-- This is now singular and only contains fields
  @ResolverField(() => User)
  author(@Parent() post: Post) {}

  @ResolverField(() => [Comment])
  comments(@Parent() post: Post) {}
}

// posts/comment.resolver.ts
@Resolver(Comment)
export class CommentResolver {
  @ResolverField(() => User)
  author(@Parent() comment: Comment) {}

  @ResolverField(() => Post)
  post(@Parent() comment: Comment) {}
}
```

Now a resolver only handles one entity - which also can be query or mutation.


### Dependencies
Another thing that comes to mind, is the dependency graph. As it is now, the users modules is dependent on the posts module (through users.comments) and the posts module is dependent on the users module (through post.auther and comment.author). Usually developers tries to avoid circular dependencies, so we should probably try to get rid of this. Users should not be dependent on posts (users could exists without posts - not vice versa). Since multiple resolvers can exist for the same entity, the UserResolver can be destructed into two resolver:

```typescript
// users/user.resolver.ts
@Resolver(User)
export class UserResolver {
  @ResolveField(() => String)
  name(@Parent() user: User) {}
}

// posts/user.resolver.ts
@Resolver(User)
export class UserResolver {
  @ResolveField(() => [Post])
  posts(@Parent() user: User) {}
}
```

Now the user.posts will be resolve through the posts module. If we for reason want to remove the posts module, we can do so without changing anything in the users module.


### Schema to code
Last if I look at the schema and want to find the implementation of a specific resolver, it is not that trivial: I have to look through all users.resolver to find the implementation of user.posts. If I have some domain knowledge, I would probably look in the posts module - that not always enough though. Taking the "one resolver, one responsibilty" to the extreme, each query, mutation or field could live in its own resolver and the resolvers could be organized into folder. Applying this, gives the following resolvers:

```typescript
// users/queries/user.query.ts
@Resolver(User)
export class UserQuery {
  @Query(() => User)
  user(@Args('id', { type: () => ID }) id: string) {}
}

// users/mutations/register-user.mutation.ts
@Resolver(User)
export class RegisterUserMutation {
  @Mutation(() => User)
  registerUser() {}
}

// users/resolvers/user.name.resolver.ts
@Resolver(User)
export class UserNameResolver {
  @ResolveField(() => String)
  name(@Parent() user: User) {}
}

// posts/queries/post.query.ts
@ArgsType()
class PostArgs {
  @Field(() => ID)
  id: string
}

@Resolver()
export class PostQuery {
  @Query(() => Post)
  post(@Args('id', { type: () => ID }) id: string) {}
}

// posts/mutations/create-post.mutation.ts
@Resolver()
export class CreatePostMutation {
  @Mutation(() => Post)
  createPost() {}
}

// posts/mutations/add-comment.mutation.ts
@Resolver()
export class MutationResolver {
  @Mutation(() => Post)
  addComment(@Args('postId', { type: () => ID }) postId: string) {}
}

// posts/resolvers/post.auther.resolver.ts
@Resolver(Post)
export class PostAuthorResolver {
  @ResolverField(() => User)
  author(@Parent() post: Post) {}
}

// posts/resolvers/post.comments.resolver.ts
@Resolver(Post)
export class PostCommentsResolver {
  @ResolverField(() => [Comment])
  comments(@Parent() post: Post) {}
}

// posts/resolvers/user.posts.resolver.ts
@Resolver(User)
export class UserPostsResolver {
  @ResolveField(() => [Post])
  posts(@Parent() user: User) {}
}
```

### Snippets
With the help from [arelstone](https://github.com/arelstone), I have created some [vscode snippets](/static/snippets/graphql-resolvers "download") for easily scaffolding these resolvers. Just create the resolver following the naming convention (`{some}.query.ts`, `{some}.mutation.ts` or some `{some-entity}.{a-field}.resovler.ts`) and use the query, mutation or resolver snippet respectively.

### Conclusion
Now each resolver have one responsibility and are easy to find from the schema (just search for `{entity}.{field}.resolver`, `{query-name}.query` or `{mutation-name}.mutation`) and the dependencies have been cleaned up. This is taken a bit to the extreme, and it is up to you as a developer to find your sweet spot - for me this worked out very well in a fairly large project (schema of 2000 lines)


