# GraphQL Transform

The GraphQL Transform provides a simple-to-use abstraction that helps us quickly create backends for our web and mobile applications on AWS. With the GraphQL Transform, we define our application's data model using the GraphQL Schema Definition Language (SDL) and the library handles converting our SDL definition into a set of fully descriptive AWS CloudFormation templates that implement our data model.

For example, we might create the backend for a blog like this:

```graphql
type Blog @model {
  id: ID!
  name: String!
  posts: [Post] @connection(name: "BlogPosts")
}

type Post @model {
  id: ID!
  title: String!
  blog: Blog @connection(name: "BlogPosts")
  comments: [Comment] @connection(name: "PostComments")
}

type Comment @model {
  id: ID!
  content: String
  post: Post @connection(name: "PostComments")
}
```

When used along with tools like the Amplify CLI, the GraphQL Transform simplifies the process of developing, deploying, and maintaining GraphQL APIs.

# Directives

# @model

Object types that are annotated with *@model* are top-level entities in the generated API. Objects annotated with *@model* are stored in Amazon DynamoDB and are capable of being protected via *@auth*, related to other objects via *@connection*, and streamed into Amazon Elasticsearch via *@searchable*. We may also apply the *@versioned* directive to instantly add a version field and conflict detection to a model type.

## Usage

Define a GraphQL object type and annotate it with the *@model* directive to store objects of that type in DynamoDB and automatically configure CRUDL and mutations.

```graphql
type Post @model {
  id: ID!
  title: String!
  tags: [String]!
}
```

We may also override the names of any generated queries, mutations, and subscriptions, or remove operations entirely.

```graphql
type Post @model(queries: {get: "post"}, mutations: null, subscriptions: null) {
  id: ID!
  title: String!
  tags: [String]!
}
```

This would create and configure a single query field **post(id: ID!): Post** and no mutation fields.

## Generates

A single *@model* directive configures the following AWS resources:

- An Amazon DynamoDB table with PAY_PER_REQUEST billing mode enabled by default
- An AWS AppSync DataSource configured to access the table above
- An AWS IAM role attached to the DataSource that allows AWS AppSync to call the above table on our behalf
- Up to 8 resolvers (create, update, delete, get, list, onCreate, onUpdate, onDelete) but this is configurable via the *queries, mutations,* and *subscriptions* arguments on the *@model* directive
- Input objects for create, update, and delete mutations
- Filter input objects that allow us to filter objects in list queries and connection fields

Sample schema document:

```graphql
type Post @model {
  id: ID!
  title: String
  metadata: Metadata
}

type MetaData {
  category: Category
}

enum Category { comedy news }
```

# @key

The *@key* directive makes it simple to configure custom index structures for *@model* types.

Amazon DynamoDB is a key-value and document database that delivers single-digit millisecond performance at any scale, but making it work for our access patterns requires a bit of forethought. DynamoDB query operations may use at most two attributes to efficiently query data. The first query argument passed to a query (the hash key) must use strict equality and the second attribute (the sort key) may use gt, ge, lt, le, eq, beginsWith and between. DynamoDB can effectively implement a wide variety of access patterns that are powerful enough for the majority of applications.

### Definition

```graphql
directive @key(fields: [String!]!, name: String, queryField: String) on OBJECT
```

### fields
A list of fields that should comprise the @key, used in conjunction with an @model type. The first field in the list will always be the **HASH** key. If two fields are provided the second field will be the **SORT** key. If more than two fields are provided, a single composite **SORT** key will be created from a combination of **fields[1...n]**. All generated GraphQL queries & mutations will be updated to work with custom **@key** directives.

### name
When provided, specifies the name of the secondary index. When omitted, specifies thet the @key is defining the primary index. We may have at most one primary key per table and therefore we may have at most one @key that does not specify a **name** per @model type.

### queryField
When defining a secondary index (by specifying the name argument), this specifies that a new top level query field that queries the secondary index should be generated with the given name.

### How to use the @key
When designing data models using the *@key* directive, the first step should be to write down our application's expected access patterns. For example, let's say we were building an e-commerce application and needed to implement access patterns like:

1. Get customers by email.
2. Get orders by customer by createdAt.
3. Get items by order by status by createdAt.
4. Get itmes by status by createdAt.

```graphql
# Get customers by email.
type Customer @model @key(fields: ["email"]) {
  email: String!
  username: String
}
```

A @key without a name specifies the key for the DynamoDB table's primary index. **We can only provide 1 @key without a new per @model type.** The example above shows the simplest case where we are specifying that the table's primary index should have a simple key where the hash key is email. This allows us to get unique customers by their email.

```graphql
query GetCustomerById {
  getCustomer(email: "me@email.com") {
    email
    username
  }
}
```

This is great for simple lookup operations, but what if we need to perform slightly more complex queries?

```graphql
# Get orders by customer by createdAt.
type Order @model @key(fields: ["customerEmail", "createdAt"]) {
  customerEmail: String!
  createdAt: String!
  orderId: ID!
}
```

This *@key* above allows us to efficiently query *Order* objects by both a *customerEmail* and the *createdAt* time stamp. The *@key* above creates a DynamoDB table where the primary index's hash key is *customerEmail* and the sort key is *createdAt*. This allows us to write queries like this:

```graphql
query ListOrdersForCustomerIn2019 {
  listOrders(customerEmail: "me@email.com", createdAt: {beginsWith: "2019"}) {
    items {
      orderId
      customerEmail
      createdAt
    }
  }
}
```

The query above shows how we can use compound key structures to implement more powerful query patterns on top of DynamoDB, but we are not quite done yet. Given that DynamoDB limits us to query by at most two attributes at a time, the @key directive helps by streamlining the process of creating composite sort keys such that we can support querying by more than two attributes at a time. For example, we can implement "Get items by order, status, and createdAt" as well as "Get items by status and createdAt" for a single *@model* with this schema.

```graphql
type Item @ model
  @key(fields: ["orderId", "status", "createdAt"])
  @key(name: "ByStatus", fields: ["status", "createdAt"], queryField: "itemsByStatus") {
  orderId: ID!
  status: Status!
  createdAt: AWSDateTime!
  name: String!
}

enum Status {
  DELIVERED
  IN_TRANSIT
  PENDING
  UNKNOWN
}
```

The primary *@key* with 3 fields performs a bit more magic than the 1 and 2 field variants. The first field orderId will be the **HASH** key as expected, but the **SORT** key will be a new composite key named *status#createdAt* that is made of the *status* and *createdAt* fields on the *@model*. The *@key* directive creates the table structures and also generates resolvers that inject composite key values for us during queries and mutations.

Using this schema, we can query the primary index to get IN_TRANSIT items created in 2019 for a given order.

```graphql
# Get items for order by status by createdAt
query ListInTransitItemsForOrder {
  listItems(orderId: "order1", statusCreatedAt: {beginsWith: {status: IN_TRANSIT, createdAt: "2019"}}) {
    items {
      orderId
      status
      createdAt
      name
    }
  }
}
```

The query above exposes the *statusCreatedAt* argument that allows us to configure DynamoDB key condition expressions without worrying about how the composite key is formed under the hood. Using the same schema, we can get all PENDING items created in 2019 by querying the secondary index "ByStatus" via the *Query.itemsByStatus*:

```graphql
query ItemsByStatus {
  itemsByStatus(status: PENDING, createdAt: {beginsWith: "2019"}) {
    items {
      orderId
      status
      createdAt
      name
    }
    nextToken
  }
}
```

## Evolving APIs with @key

There are a few important things to think about when making changes to APIs using *@key*. When we need to enable a new access pattern or change an existing access pattern we should follow these steps.

1. Create a new index that enables the new or updated access pattern.
2. If adding a @key with 3 or more fields, we will need to back-fill the new composite sort key for existing data. With a *@key(fields: ["email", "status", "date"])*, we would need to backfill the *status#date* field with composite key vaules made up of each object's *status* and *date* fields joined with a *#*. We do not need to backfill data for *@key* directives with 1 or 2 fields.
3. Deploy our additive changes and update any downstream applications to use the new access pattern.
4. Once we are certain taht we do not need the old index, remove its *@key* and deploy the API again.

# @connection

The *@connection* directive enables us to specify relationships between *@model* types. Currently, this supports one-to-one, one-to-many, and many-to-one relationships. We may implement many-to-many relationships using two one-to-many connections and a joining *@model* type. See the usage section for details.

## Definition

```graphql
directive @connection(keyName: String, fields: [String!]) on FIELD_DEFINITION
```

## Usage

Relationships between types are specified by annotating fields on an *@model* object type with the *@connection* directive.

The *fields* argument can be provided and indicates which fields can be queried by to get connected objects. The *keyName* argument can optionally be used to specify the name of secondary index (an index that was set up using *@key*) that should be queried from the other type in the relationship.

When specifying a *keyName*, the *fields* argument should be provided to indicate with field(s) will be used to get connected objects. If *keyName* is not provided, then *@connection* queries the target table's primary index.