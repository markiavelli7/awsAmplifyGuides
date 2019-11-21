# Configuring Resolvers

When we use GraphQL to generate our resolvers, we can go into the /amplify/backend/api/myapp/build/resolvers folder and inspect our resolvers:

**Category.products.req.vtl**

```vtl
#set( $limit = $util.defaultIfNull($context.args.limit, 10) )
#set( $query = {
  "expression": "#connectionAttribute = :connectionAttribute",
  "expressionNames": {
      "#connectionAttribute": "categoryProductsId"
  },
  "expressionValues": {
      ":connectionAttribute": {
          "S": "$context.source.id"
    }
  }
} )
{
  "version": "2017-02-28",
  "operation": "Query",
  "query":   $util.toJson($query),
  "scanIndexForward":   #if( $context.args.sortDirection )
    #if( $context.args.sortDirection == "ASC" )
true
    #else
false
    #end
  #else
true
  #end,
  "filter":   #if( $context.args.filter )
$util.transform.toDynamoDBFilterExpression($ctx.args.filter)
  #else
null
  #end,
  "limit": $limit,
  "nextToken":   #if( $context.args.nextToken )
"$context.args.nextToken"
  #else
null
  #end,
  "index": "gsi-Category.products"
}
```

**Category.products.res.vtl**

```vtl
#if( !$result )
  #set( $result = $ctx.result )
#end
$util.toJson($result)
```

For each resolver, there is a request (req) and response (res) template. 

In GraphQL, when the browser sends a query to the API, we need to have a resolver to handle the operation.

We can have resolvers that use the DynamoDB tables to resolve the query. Or we can write a Lambda function that goes to the DynamoDB table and gets a result and use the return value of the Lambda function as data for our resolver. 



In the AWS AppSync console, we can go to the **Schema** page. In the **Query** type on the right side, choose *Attach resolver* next to the *getTodos* field. On the **Create Resolver** page, we can choose the data source we just created, and then choose a default template or paste in our own. For common use cases, the AWS AppSync console has built-in templates that we can use for getting items from data sources (for example, all item queries, individual lookups, etc.).

```json
{
  "version": "2017-02-28",
  "operation": "Scan"
}
```

We always need a response mapping template. The console provides a default with the following passthrough value for lists:

```javascript
$util.toJson($context.result.items)
```

In this example, the *context* object (aliased as *$ctx*) for lists of items has the form  *$context.result.items*. If our GraphQL operation returns a single item, it would be *$context.result*. AWS AppSync provides helper functions for common operations, such as the *util.toJson* function listed previously, to format responses properly.

**Note:** The default resolver we have just created is a **Unit** resolver, but AppSync also supports creating **Pipeline** resolvers to run operations against multiple data sources in sequence.

## Adding a Resolver for Mutations

We can repeat the preceding process, starting at the **Schema** page and choosing the **Attach resolver** for the *addTodo* mutation. Because this is a mutation where we're adding a new item to DynamoDB, we can use the following request mapping template:

```json
{
  "version": "2017-02-28",
  "operation": "PutItem"
  "key": {
    "id": {"S": "${ctx.args.id}"}
  },
  "attributeValues": $util.dynamodb.toMapValuesJson($ctx.args)
}
```

AWS AppSync automatically converts arguments defined in the addTodo field from our GraphQL schema into DynamoDB operations. The above example stores the records in DynamoDB using a key of *id*, which is passed through from the mutation argument as $ctx.args.id. All of the other fields we pass through are automatically mapped to DynamoDB attributes with $util.dynamodb.toMapValuesJson($ctx.args).

For this resolver, we use the following response mapping template:

```javascript
$utils.toJson($context.result)
```