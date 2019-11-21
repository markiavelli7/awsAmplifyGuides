# @model

```graphql
type Todo @model {
  id: ID!
  name: String!
  body: String
  completed: Boolean
}
```

As soon as AppSync sees the @model directive on the Todo type, it is going to create a DynamoDB table with the GraphQL field name and types that we provided,do all of the linking, and provide all of the DynamoDB operations.

# @connection

The @connection directive enables us to specify relationships between @model object types. Currently, this supports one-to-one, one-to-many, and many-to-one relationships. We may implement many-to-many relationships ourself using two one-to-many connections and joining @model type.

## Definition

```graphql
directive @connection(name: String, keyField: String, sortField: String, limit: Int) on FIELD_DEFINITION
```

## Usage

Relationships between data are specified by annotating fields on an *@model* object type with the *@connection* directive. We can use the *keyField* to specify what field should be used to partition the elements within the index and the *sortField* argument to specify how the records should be sorted.

**Unnamed Connections**

## One-to-One Connection

In the simplest case, we can define a one-to-one connection:

```graphql
type Project @model {
  id: ID!
  name: String
  team: Team @connection
}

type Team @model {
  id: ID!
  name: String!
}
```

After it's transformed, we can create projects with a team as follows:

```graphql
mutation CreateProject {
  createProject(input: {name: "New Project", projectTeamId: "a-team-id"}) {
    id
    name
    team {
      id
      name
    }
  }
}
```

## One-to-Many Connection

```graphql
type Post @model {
  id: ID!
  title: String!
  comments: [Comment] @connection
}

type Comment @model {
  id: ID!
  content: String!
}
```

After it's transformed, we can create comments with a post as follows:

```graphql
mutation CreateCommentOnPost {
  createComment(input: {content: "A comment", postCommentsId: "a-post-id"}) {
    id
    content
  }
}
```

**Note** The postCommentsId field on the input may seem unusual. In the one-to-many case without a provided *name* argument there is only partial information to work with, which results in the unusual name. To fix this, provide a value for the *@connection*'s name argument and complete the bi-directional relationship by adding a corresponding *@connection* field to the **Comment** type.

**Named Connections**

The **name** argument specifies a name for the connection and it's used to create bi-directional relationships that reference the same underlying foreign key.

For example, if we wanted our Post.comments and Comment.post fields to refer to opposite sides of the same relationship, we need to provide a name.

```graphql
type Post @model {
  id: ID!
  title: String!
  comments: [Comment] @connection(name: "PostComments", sortField: "createdAt")
}

type Comment @model {
  id: ID!
  content: String!
  post: Post @connection(name: "PostComments", sortField: "createdAt")
  createdAt: String
}
```

After it's transformed, create comments with a post as follows:

```graphql
mutation CreateCommentOnPost {
  createComment(input: {content: "A comment", commentPostId: "a-post-id"}) {
    id
    content
    post {
      id
      title {
        comments {
          id
          # and so on...
        }
      }
    }
  }
}
```

When we query the connection, the comments will return sorted by their *createdAt* field

```graphql
query GetPostAndComments {
  getPost(id: "...") {
    id
    title
    comments {
      items {
        content
        createdAt
      }
    }
  }
}
```

**Many-To-Many Connections**

We can implement many-to-many ourself using two one-to-many @connections and a joining @model. For example:

```graphql
type Post @model {
  id: ID!
  title: String!
  editors: [PostEditor] @connection(name: "PostEditors")
}

# Create a join model and disable queries as we don't need them
# and can query through Post.editors and User.posts
type PostEditor @model(queries: null) {
  id: ID!
  post: Post! @connection(name: "PostEditors")
  editor: User! @connection(name: "UserEditors")
}

type User @model {
  id: ID!
  username: String!
  posts: [PostEditor] @connection(name: "UserEditors")
}
```

We can then create Posts & Users independently and join them in a many-to-many by creating PostEditor objects.

## Limit

The default number of nested objects is 10. We can override this behavior by setting the limit argument:

```graphql
type Post {
  id: ID!
  title: String!
  comments: [Comment] @connection(limit: 50)
}

type Comment {
  id: ID!
  content: String!
}
```

## Generates

In order to keep connection queries fast and efficient, the GraphQL transform manages global secondary indexes (GSIs) on the generated tables on our behalf.

**Note** The *@connection* directive manages these GSIs under the hood but there are limitations to be aware of.After we have pushed a @connection directive, we should not try to change it. If we try to change it, the DynamoDB UpdateTable will fail due to one of a set of service limitations around changing GSIs. Should we need to change a *@connection*, we should add a new *@connection* that implements the new access pattern, update our application to use the new *@connection*, and then delete the old *@connection* when it's no longer needed.