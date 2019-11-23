## One-to-One Relationship

```graphql
type Department @model {
  id: ID!
  name: String
  manager: Employee @connection
}

type Employee @model {
  id: ID!
  name: String
  age: Int
}
```

To make a one-to-one relationship (Department has one manager, which is an Employee), we use the @connection AppSync directive. We make a connection on the manager field to the Employee type.

```graphql
  manager: Employee @connection
```

When Amplify compiles our schema and sees the Department type with the @model directive, it's going to create the Department table in DynamoDB with an id, name, and manager attributes. It is also going to create another reference for the manager by adding an attribute called #managerId. So the Department table contains a reference to the #managerId.

## One-to-Many Relationship

```graphql
type Department @model {
  id: ID!
  name: String
  manager: Employee @connection
  employees: [Employee]! @connection(name: "DepartmentEmployees")
}

type Employee @model {
  id: ID!
  name: String
  age: Int
  department: Department @connection(name: "DepartmentEmployees")
}
```

In this case, we have a one-to-many relationship (a Department can have many Employees). When we create an employee in the employee table, we should have an attribute called departmentId
# AppSync Schema Design

Schema files are text files, usually named schema.graphql. We can create this file and submit it to AWS AppSync by using the CLI or navigating to the console and adding the following under the Schema page:

```graphql
schema {
}
```

Every schema has this root for processing. This fails to process until we add a root query type.

## Adding a Root Query Type

For this example, we create a Todo application. A GraphQL schema must have a root query type, so we add a root type named Query with a single getTodos field that returns a list containing Todo objects:

```graphql
schema {
  query: Query
}

type Query {
  getTodos: [Todo]
}
```

## Defining a Todo Type

Now, we can create a type that contains the data for a Todo object:

```graphql
schema {
  query: Query
}

type Query {
  getTodos: [Todo]
}

type Todo {
  id: ID!
  name: String
  description: String
  priority: Int
}
```

The *Todo* object type has fields that are scalar types, such as strings and intergers. AWS AppSync also has enhanced Scalar types in addition to the base GraphQL scalars that we can use in a schema. Any field that ends in an exclamation point is a non-nullable (must always return a value when we query) field. The ID scalar type is a unique identifier that can either be a String or an Int. We can control these in our resolver mapping templates for automatic assignment.

## Adding a Mutation Type

Now that we have an object type and can query the data, if we want to add, update, or delete data via the API, we need to add a mutation type to our schema:

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getTodos: [Todo]
}

type Mutation {
  addTodo(input: TodoInput): Todo
}

type Todo {
  id: ID!
  name: String
  description: String
  priority: Int
}

input TodoInput {
  id: ID!
  name: String
  description: String
  priority: Int
}
```

## Modifying the Todo Example with a Status

At this point, our GraphQL API is functioning structurally for reading and writing Todo objects--it just doesn't have a data source. We can modify this API with more advanced functionality, such as adding a status to our Todo, which comes from a set of values defined as an ENUM

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getTodos: [Todo]
}

type Mutation {
  addTodo(input: TodoInput): Todo
}

type Todo {
  id: ID!
  name: String
  description: String
  priority: Int
  status: TodoStatus
}

enum TodoStatus {
  done
  pending
}

input TodoInput {
  id: ID!
  name: String!
  description: String!
  priority: Int!
  status: TodoStatus!
}
```

An *ENUM* is like a string, but it can take one of a set of values. Above, we added this type, modified the Todo type, and added the Todo field to contain this functionality.

## Subscriptions

Subscriptions in AWSAppSync are invoked as a response to a mutation. We can configure this with a Subscription type and the @subscribe() directive in the schema to denote which mutations invoke one or more subscriptions.

## Relations and Pagination

Suppose we had a million todos. We wouldn't want to fetch all of these every time, instead, we would want to paginate through them. We would want to make the following changes to our schema:

- To get the getTodos field, we would add two input arguments: *limit* and *nextToken*
- Add a new TodoConnection type that has todos and nextToken fields
- Change getTodos so that it returns TodoConnection (not a list of Todos)

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getTodos(limit: Int, nextToken: String): TodoConnection
}

type Mutaion {
  addTodo(input: TodoInput): Todo
}

type Todo {
  id: ID!
  name: String
  description: String
  priority: Int
  status: TodoStatus
}

type TodoInput {
  id: ID!
  name: String!
  description: String
  priority: Int
  status: TodoStatus
}

type TodoConnection {
  todos: [Todo]
  nextToken: String
}

enum TodoStatus {
  done
  pending
}
```

The TodoConnection type allows us to return a list of *todos* and a *nextToken* for getting the next batch of *todos*. Note, that it is a single *TodoConnection* and not a list of connections. Inside the connection is a list of todo items (`[Todo]`) which gets returned with a pagination token. In AWSAppSync, this is connected to Amazon DynamoDB with a mapping template and automatically generated as an encrypted token. This converts the value of the *limit* argument to the *maxResults* parameter and the *nextToken* argument to the *exclusiveStartKey* parameter.

Next, suppose our todos have comments, and we want to run a query that returns all the comments for a *todo*. This is handled through *GraphQL* connections, as we created in the previous schema. We can modify our schema to have a *Comment* type, add a *comments* field to the *Todo* type, and add an *addComment* field on the *Mutation* type as follows:

```graphql
schema {
  query: Query
  mutation: Mutation
}

type Query {
  getTodos(limit: Int, nextToken: String): TodoConnection
}

type Mutation {
  addTodo(input: TodoInput): Todo
  addComment(input: CommentInput): Comment
}

type Comment {
  todoid: ID!
  commentid: String!
  content: String
}

input CommentInput {
  todoid: ID!
  content: String
}

type Todo {
  id: ID!
  name: String
  description: String
  priority: Int
  status: TodoStatus
  comments: [Comment]
}

input TodoInput {
  id: ID!
  name: String
  description: String
  priority: Int
  status: TodoStatus
}

type TodoConnection {
  todos: [Todo]
  nextToken: String
}

enum TodoStatus {
  done
  pending
}
```

Note that the *Comment* type has the *todoid* that it's associated with, *commentid*, and *content*. This corresponds to a primary key + sort key combination in the Amazon DynamoDB table we create later.

The application graph on top of our existing data sources in AWS AppSync enables us to return data from two separate data sources in a single GraphQL query. In the example, the assumption is that there is both a Todos table and a Comments table.
