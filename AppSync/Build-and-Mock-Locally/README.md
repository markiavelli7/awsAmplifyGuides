# Creating the API

```javascript
amplify add api
```

*Note:* Choosing Amazon Cognito User Pool will allow us to use the Authentication that we just created to interact with our API.

*Also:* If we want to add general reading abilities to our users, we want to add an API Key from the advanced configuration settings.

For the API key name, we want to give it the name publicaccess

## Editing the Schema

*schema.graphql*
```graphql
type Category @model(subscriptions: null)
  @auth(rules: [
    {allow: groups, groups: ["Admin"], operations: [create, update, delete]},
    {allow: public, operations: [read]}
  ]) {
    id: ID!
    name: String!
    products: [Product] @connection
  }

  type Product @model(subscriptions: null)
    @auth(rules: [
      {allow: groups, groups: ["Admin"], operations: [create, update, delete]},
      {allow: public, operations: [read]}
    ]) {
      id: ID!
      name: String!
      price: Float!
    }

    type Query {
      calculate(price: Float!, location: String!): PriceInfo @function(name: "calculateTax-${env}")
    }

    type PriceInfo {
      calculateTax: Float
      finalPrice: Float
    }
```

The @connection directive

```graphql
products: [Product] @connection
```

is going to draw a connection/relationship between the Product and Category Types. This lets us create a new Product and associate it with one of the existing Categories.

We use the @auth directive

```graphql
@auth(rules: [
    {allow: groups, groups: ["Admin"], operations: [create, update, delete]},
    {allow: public, operations: [read]}
  ])
```

to specify our authorization rules. For our api, we want to give anyone read and write access so we set allow to public and the operation of read.

For anyone that is able to create, update, or delete Products or Categories, we want to make sure that they are part of an admin group. To do that, we set allow to groups and pass in a hardcoded name of "Admin" into our groups array, and for operations, we are allowing create, update, and, delete. Instead of hard-coding "Admin" into groups, this could be dynamically read off of a resolved field in our GraphQL type.

We also have a type of Query that we are associating with the Lambda function that we created:

```graphql
type Query {
      calculate(price: Float!, location: String!): PriceInfo @function(name: "calculateTax-${env}")
    }

    type PriceInfo {
      calculateTax: Float
      finalPrice: Float
    }
```

So, we can either hit our Lambda function directly, or we can use it within GraphQL.

calculate is a resolver that takes a price and location as arguments. This resolver will return a PriceInfo object that we define with a Type. The PriceInfo object will contain two fields, calculateTax which takes a Float and finalPrice, which is also a Float.