# AppSync Defined Scalars

- AWSDate
- AWSTime
- AWSDateTime
- AWSTimestamp
- AWSEmail
- AWSJSON
- AWSURL
- AWSPhone
- AWSIPAddress

## Example Schema Usage

```graphql
type Mutation {
  putObject(
    email: AWSEmail,
    json: AWSJSON,
    date: AWSDate,
    time: AWSTime,
    datetime: AWSDateTime,
    timestamp: AWSTimestamp,
    url: AWSURL,
    phoneno: AWSPhone,
    ip: AWSIPAddress
  ) : Object
}

type Object {
  id: ID!
  email: AWSEmail
  json: AWSJSON
  date: AWSDate
  time: AWSTime
  datetime: AWSDateTime
  timestamp: AWSTimestamp
  url: AWSURL
  phoneno: AWSPhone
  ip: AWSIPAddress
}

type Query {
  getObject(id: ID!): Object
  listObjects: [Object]
}

schema {
  query: Query
  mutation: Mutation
}
```

The following **request** template is for *putObject*:

```json
{
  "version": "2017-02-28",
  "operation": "PutItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
  },
  "attributeValues": $util.dynamodb.toMapValuesJson($ctx.args)
}
```

The response template for *putObject* will be:

```javascript
$util.toJson($ctx.result)
```

We can use the following **request** template for *getObject*:

```json
{
  "version": "2017-02-28",
  "operation": "GetItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($ctx.args.id),
  }
}
```

The response template for *getObject* will be:

```javascript
$util.toJson($ctx.result)
```

We can use the following **request** template for *listObjects*:

```json
  "version": "2017-02-28",
  "operation": "Scan",
```

The response template for *listObjects* will be:

```javascript
$util.toJson($ctx.result.items)
```

Examples of using this schema with GraphQL queries are below:

```graphql
mutation CreateObject {
  putObject(
    email: "naidabailey@amazon.com"
    json: "{\"a\":1, \"b\":3, \"string\":234}"
    date: "1970-01-01Z"
    time: "12:00:34."
    datetime: "1930-01-01T16:00:00-07:00"
    timestamp: -123123
    url: "http://amazon.co.in"
    phoneno: "+91 704-011-2342"
    ip: "127.0.0.1/8"
  ) {
    id
    email
    json
    date
    time
    datetime
    url
    timestamp
    phoneno
    ip
  }
}

query getObject {
  getObject(id:"0d97daf0-48e6-4ffc-8d48-0537e8a843d2") {
    email
    url
    timestamp
    phoneno
    ip
  }
}

query listObjects {
  listObjects {
    json
    date
    time
    datetime
  }
}
```