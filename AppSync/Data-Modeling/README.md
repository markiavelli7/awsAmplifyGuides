# Understanding the GraphQL Directives

**@model** - Generates a NoSQL database, resolvers, and CRUD + List + subscription GraphQL operation definitions from a base GraphQL type.

**@connection** - Enables a relationships between GraphQL types

**@key** - Enables effectient queries with conditions by using underlying database index structures for optimization

Let's say that we have basic GraphQL types that look like this:

```graphql
type Book {
  id: ID!
  title: String
  author: Author
}

type Author {
  id: ID!
  name: String!
}
```

To first expand this into a full API with a database, resolvers, CRUD + List operations, and subscriptions, we can add the @model directive to each type:

```graphql

type Book @model {
  id: ID!
  title: String
  author: Author
}

type Author @model {
  id: ID!
  name: String!
}
```

Next, we want to add a relationship between the book and the author. To do this, we can use the @connection directive:

```graphql
type Book @model {
  id: ID!
  title: String
  author: Author @connection
}

type Author @model {
  id: ID!
  name: String!
}
```

Next, let's say that we want to query the books by the book title. How could we manage this? We could update the Book type with a *@key* directive:

```graphql
type Book 
  @model
  @key(name: "byTitle", fields: ["title"], queryField: "bookByTitle") {
    id: ID!
    title: String
    author: Author @connection
  }
```

Now we can use the following query to query by book title:

```graphql
query byTitle {
  bookByTitle(title: "Grapes of Wrath") {
    items {
      id
      title
    }
  }
}
```

The @key directive in the above example takes the following arguments:

```graphql
# name - name of the key
# fields - field(s) that we will be querying by
# queryField - name of the GraphQL query to be generated

@key(name: "byTitle", fields: ["title"], queryField: "bookByTitle")
```

We can take this one step further. What if we want to create a *Publisher* type and assign books to a certain publisher? We want to use our existing *Book* type and associate it with our new *Publisher* type:

```graphql
type Publisher @model {
  name: String!
  id: ID!
  books: [Book] @connection(keyName: "byPublisherId", fields: ["id"])
}

type Book @model
  @key(name: "byTitle", fields: ["title"], queryField: "bookByTitle")
  @key(name: "byPublisherId", fields:["publisherId"], queryField: "booksByPublisherId") 
  {
    id: ID!
    publisherId: ID!
    title: String
    author: Author @connection
  }
```

Here, we're added a new @key directive named *byPublisherId* and associated it with the resolved *books* field of the *Publisher* type. Now, we can query publishers and also get the associated books and authors:

```graphql
query listPublishers {
  items {
    id {
      books {
        items {
          id
          title
          author {
            id
            name
          }
        }
      }
    }
  }
}
```

Furthermore, with the new booksByPublisherId query, we can also directly query all books by publisher ID:

```graphql
query booksByPublisherId($publisherId: ID!) {
  booksByPublisherId(publisherId: $publisherId) {
    items {
      id
      title
    }
  }
}
```

## 17 Access Patterns

This example has the following types:
- Warehouse
- Product
- Inventory
- Employee
- AccountRepresentative
- Customer
- Order

Here are the access patterns that we'll be implementing:

1. Look up employee details by Employee ID
2. Query employee details by employee name
3. Find an employee's phone number(s)
4. Find a customer's phone number(s)
5. Get orders for a given customer within a given date range
6. Show all open orders wihin a given date range across all customers
7. See all employees recently hired
8. Find all employees working in a given warehouse
9. Get all items on order for a given product
10. Get current inventories for a given product at all warehouses
11. Get customers by account representative
12. Get orders by account representative and date
13. Get all items on order for a given product
14. Get all employees with a given job title
15. Get inventory by product and warehouse
16. Get total product inventory
17. Get account representatives ranked by order total and sales period

The following schema introduces the required keys and connections so that we can support 17 access patterns.

```graphql
type Order @model
  @key(name: "byCustomerByStatusByDate", fields: ["customerID", "status", "date"])
  @key(name: "byCustomerByDate", fields: ["customerID", "date"])
  @key(name: "byRepresentativeByDate", fields: ["accountRepresentativeID", "date"])
  @key(name: "byProduct", fields: ["productID", "id"])
{
  id: ID!
  customerID: ID!
  accountRepresentativeID: ID!
  productID: ID!
  status: String!
  amount: Int!
  date: String!
}

type Customer @model
  @key(name: "byRepresentative", fields: ["accountRepresentativeID", "id"])
{
  id: ID!
  name: String!
  phoneNumber: String
  accountRepresentativeID: ID!
  ordersByDate: [Order] @connection(keyName: "byCustomerByDate", fields: ["id"])
  ordersByStatusDate: [Order] @connection(keyName: "byCustomerByStatusByDate", fields: ["id"])
}

type Employee @model
  @key(name: "newHire", fields: ["newHire", "id"], queryField: "employeeNewHire")
  @key(name: "newHireByStartDate", fields: ["newHire", "startDate"], queryField: "employeesNewHireByStartDate")
  @key(name: "byName", fields: ["name", "id"], queryField: "employeeByName")
  @key(name: "byTitle", fields: ["jobTitle", "id"], queryField: "employeesByJobTitle")
  @key(name: "byWarehouse", fields: ["warehouseID", "id"])
{
  id: ID!
  name: String!
  startDate: String!
  phoneNumber: String!
  warehouseID: ID!
  jobTitle: String!
  newHire: String! # We have to use String type, because Boolean types cannot be sort keys
}

type Warehouse @model {
  id: ID!
  employees: [Employee] @connection(keyName: "byWarehouse", fields: ["id"])
}

type AccountRepresentative @model
  @key(name: "bySalesPeriodByOrderTotal", fields: ["salesPeriod", "orderTotal"], queryField: "repsByPeriodAndTotal")
{
  id: ID!
  customers: [Customer] @connection(keyName: "byRepresentative", fields: ["id"])
  orders: [Order] @connection(keyName: "byRepresentativeByDate", fields: ["id"])
  orderTotal: Int
  salesPeriod: String
}

type Inventory @model
  @key(name: "byWarehouseID", fields: ["warehouseID"], queryFields: "itemsByWarehouseID")
  @key(fields: ["productID", "warehouseID"])
{
  productID: ID!
  warehouseID: ID!
  inventoryAmount: Int!
}

type Product @model {
  id: ID!
  name: String!
  orders: [Order] @connection(keyName: "byProduct", fields: ["id"])
  inventories: [Inventory] @connection(fields: ["id"])
}
```

Now that we have the schema created, we can create the items in the database that we will be operating against:

```graphql
# first
mutation createWarehouse {
  createWarehouse(input: {id: "1"}) {
    id
  }
}

# second
mutation createEmployee {
  createEmployee(input: {
    id: "amanda",
    startDate: "2018-05-22",
    phoneNumber: "6015555555",
    warehouseID: "1",
    jobTitle: "Manager",
    newHire: "true"
  }) {
    id jobTitle name newHire phoneNumber startDate warehouseID
  }
}

# third
mutation createAccountRepresentative {
  createAccountRepresentative(input: {
    id: "dabit",
    orderTotal: 400000,
    salesPeriod: "January 2019"
  }) {
    id orderTotal salesPeriod
  }
}

# fourth
mutation createCustomer {
  createCustomer(input: {
    id: "jennifer_thomas",
    accountRepresentativeID: "dabit",
    name: "Jennifer Thomas",
    phoneNumber: "+16015555555"
  }) {
    id name accountRepresentativeID phoneNumber
  }
}

# fifth
mutation createProduct {
  createProduct(input: {
    id: "yeezyboost",
    name: "Yeezy Boost"
  }) {
    id
    name
  }
}

# sixth
mutation createInventory {
  createInventory(input: {
    productID: "yeezyboost",
    warehouseID: "1",
    inventoryAmount: 300
  }) {
    id productID inventoryAmount warehouseID
  }
}

# seventh
mutation createOrder {
  createOrder(input: {
    amount: 300
    date: "2018-07-12"
    status: "pending"
    accountRepresentativeID: "dabit"
    customerID: "jennifer_thomas"
    productID: "yeezyboost"
  }) {
    id customerID accountRepresentativeID amount date customerID productID
  }
}
```

## 1. Look up employee details by Employee ID:

This can simply be done by querying the employee model with an employee ID, no *@key* or *@connection* is needed to make this work:

```graphql
query getEmployee($id: ID!) {
  getEmployee(id: $id) {
    id
    name
    phoneNumber
    startDate
    jobTitle
  }
}
```

## 2. Query employee details by employee name:

The *@key byName* on the *Employee* type makes this access-pattern feasible because under the covers, an index is created and a query is used to match against the name field. We can use this query:

```graphql
query employeeByName($name: String) {
  employeeByName(name: $name) {
    items {
      id name phoneNumber startDate jobTitle
    }
  }
}
```

## 3. Find an Employee's phone number:

Either one of the previous queries would work to find an employee's phone number as long as one has their ID or name.

## 4. Find a customer's phone number:

A similar query to those given above , but on the Customer model would give a customer's phone number.

```graphql
query getCustomer($customerID: ID!) {
  getCustomer(id: $customerID) {
    phoneNumber
  }
}
```

## 5. Get orders for a given customer within a given data range:

There is a one-to-many relation that lets all the orders of a customer be queried.

This relationship is created by having the *@key* name *byCustomerByDate* on the Order model that is queried by the connection on the orders field of the Customer model.

A sort key with the date is used. What this means is that the GraphQL resolver can use predicates like *Between* to efficiently search the date range rather than scanning all records in the database and then filtering them out.

The query one would need to get the oders to a customer within a date range would be:

```graphql
query getCustomerWithOrdersByDate($customerID: ID!) {
  getCustomer(id: $customerID) {
    ordersByDate(date: {
      between: ["2018-01-22", "2020-10-11"]
    }) {
      items {
        id
        amount
        productID
      }
    }
  }
}
```

## 6. Show all open orders within a given date range across all customers:

The *@key byCustomerByStatusByDate* enables us to run a query that would work for this access pattern.

In this example, a composite sort key (combination of two or more keys) with the *status* and *date* is used. What this means is that the unique identifier of a record in the database is created by concatenating these two fields (status and date) together, and then the GraphQL resolver can use predicates like *Between* or *Contains* to efficiently search the unique identifier for matches rather than scanning all records in the database and then filtering them out.

```graphql
query getCustomerWithOrdersByStatusDate($customerID: ID!) {
  getCustomer(id: $customerID) {
    ordersByStatusDate(statusDate: {
      between: [
        {status: "pending", date: "2018-01-22"},
        {status: "pending", date: "2020-10-11"}
      ]
    }) {
      items {
        id
        amount
        date
      }
    }
  }
}
```

## 7. See all employees hired recently:

Having *@key(name: "newHire", fields:["newHire", "id"])* on the *Employee* model allows one to query by whether an employee has been hired recently.

```graphql
query employeesNewHire {
  employeesNewHire(newHire: "true") {
    items {
      id
      name
      phoneNumber
      startDate
      jobTitle
    }
  }
}
```

We can also query and have the results returned by start date by using the *employeesNewHireByStartDate* query:

```graphql
query employeesNewHireByDate {
  employeesNewHireByStartDate(newHire: "true") {
    items {
      id
      name
      phoneNumber
      startDate
      jobTitle
    }
  }
}
```

## 8. Find all employees working in a given warehouse:

This needs a one to many relationship from warehouses to employees. As can be seen from the *@connection* in the *Warehouse* model, this connection uses the *byWarehouse* key on the *Employee* model. The relevant query would look like this:

```graphql
query getWarehouse($warehouseID: ID!) {
  getWarehouse(id: $warehouseID) {
    id
    employees {
      items {
        id
        name
        startDate
        phoneNumber
        jobTitle
      }
    }
  }
}
```

## 9. Get all items on order for a given product:

This access-pattern would use a one-to-many relation from products to orders. With this query, we can get all orders of a given product:

```graphql
query getProductOrders($productID: ID!) {
  getProduct(id: $productID) {
    id
    orders {
      items {
        id
        status
        amount
        date
      }
    }
  }
}
```

## 10. Get current inventories for a product at all warehouses:

The query needed to get the inventories of a product in all warehouses would be:

```graphql
query getProductInventoryInfo($productID: ID!) {
  getProduct(id: $productID) {
    id
    inventories {
      items {
        warehouseID
        inventoryAmount
      }
    }
  }
}
```

## 11. Get customers by account representative:

this uses a one-to-many connection between account representatives and customers:

```graphql
query getCustomersForAccountRepresentative($representativeId: ID!) {
  getAccountRepresentative(id: $representativeId) {
    customers {
      items {
        id
        name
        phoneNumber
      }
    }
  }
}
```

## 12. Get orders by account representative and date:

As can be seen in the *AccountRepresentative* model this connection uses the *byRepresentativeByDate* field on the *Order* model to create the connection needed. The query needed would look like this:

```graphql
query getOrdersForAccountRepresentative($representativeId: ID!) {
  getAccountRepresentative(id: $representativeId) {
    id
    orders(date: {
      between: [
        "2010-01-22", "2020-10-11"
      ]
    }) {
      items {
        id
        status
        amount
        date
      }
    }
  }
}
```

## 13. Get all items on order for a given product:

This is the same as number 9.

## 14. Get all employees with a given job title:

Using the *byTitle @key* makes this access pattern quite easy.

```graphql
query employeesByJobTitle {
  employeesByJobTitle(jobTitle: "Manager") {
    items {
      id
      name
      phoneNumber
      jobTitle
    }
  }
}
```

## 15. Get inventory by product by warehouse:

Here, having the inventories be held in a separate model is particularly useful since this model can have its own partition key and sort key such that the inventories themselves can be queried as is needed for this access-pattern.

A query on this model would look like this:

```graphql
query inventoryByProductAndWarehouse($productID: ID!, $warehouseID: ID!) {
  getInventory(productID: $productID, warehouseID: $warehouseID) {
    productID
    warehouseID
    inventoryAmount
  }
}
```

We can also get all inventory from an individual warehouse by using the *itemsByWarehouseID* query created by the *byWarehouseID* key:

```graphql
query byWarehouseId($warehouseID: ID!) {
  itemsByWarehouseID(warehouseID: $warehouseID) {
    items {
      inventoryAmount
      productID
    }
  }
}
```

## 16. Get total product inventory:

How this would be done depends on the use case. If one just wants a list of all inventories in all warehouses, one could just run a list inventories on the Inventory model:

```graphql
query listInventories {
  listInventories {
    items {
      productID
      warehouseID
      inventoryAmount
    }
  }
}
```

## 17. Get sales representatives ranked by order total and sales period:

It's uncertain exactly what this means. One take is that the sales period is either a date range or maybe even a month or week. Therefore, we can set the salse period as a string and query using the combination of *salesPeriod* and *orderTotal*. We can also set the *sortDirection* in order to get the return values from largest to smallest:

```graphql
query repsByPeriodAndTotal {
  repsByPeriodAndTotal(
    sortDirection: DESC,
    salesPeriod: "January 2019",
    orderTotal: {
      ge: 1000
    }
  ) {
    items {
      id
      orderTotal
    }
  }
}
```

# Additional Basic Access Patterns

Since we are using the GraphQL Transform library, we will be getting all of the basic read & list operations on each type as well. So for each type, we will have a *get* and *list* operation.

So for **Order, Customer, Employee, Warehouse, AccountRepresentative, Inventory** , and **Product** we can also perform basic *get* by ID and *list* operations:

```graphql
query getOrder($id: ID!) {
  getOrder(id: $id) {
    id
    customerID
    accountRepresentativeID
    productID
    status
    amount
    date
  }
}

query listOrders {
  listOrders {
    items {
      id
      customerID
      accountRepresentativeID
      productID
      status
      amount
      date
    }
  }
}
```

