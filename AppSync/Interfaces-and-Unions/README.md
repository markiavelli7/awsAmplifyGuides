# Interfaces and Unions in GraphQL

## Interfaces

We could represent an *Event* interface that represents any kind of activity or gathering of people. Possible kinds of events are Concert, Conference, and Festival. These types all share common characteristics, including a name, a venue where the event is taking place, and a start and end date. These types also have differences, a *Conference* offers a list of speakers and workshops while a *Concert* features a performing band.

**Event Interface**

```graphql
interface Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestrictions: Int
}
```

And each of the types implements the Event interface is as follows:

```graphql
type Concert implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  performingBand: String
}

type Festival implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  performers: [String]
}

type Conference implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  speakers: [String]
  workshops: [String]
}
```

Interfaces are useful to represent elements that might be of several types. For example, we could search for all events happening at a specific venue. Let's add a findEventsByVenue field to the schema as follows:

```graphql
schema {
  query: Query
}

type Query {
  # Retrieve Events at a specific Venue
  findEventsAtVenue(venueId: ID!): [Event]
}

type Venue {
  id: ID!
  name: String
  address: String
  maxOccupancy: Int
}

interface Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestrictions: Int
}

type Concert implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestrictions: Int
  performingBand: String
}

type Festival implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestrictions: Int
  performers: [String]
}

type Conference implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestrictions: Int
  speakers: [String]
  workshops: [String]
}
```

*findEventsByVenue* returns a list of *Event*. Because GraphQL interface fields are common to all the implementing types, it's possible to select any fields on the *Event* interface (*id*, *name*, *startsAt*, *endsAt*, *venue*, and *minAgeRestriction*). Additionally, we can access the fields on any implementing type using GraphQL fragments, as longs as we specify the type.

A GraphQL query that uses the interface:

```graphql
query {
  findEventsAtVenue(venueId: "Madison Square Garden") {
    id
    name
    minAgeRestriction
    startsAt

    ...on Festival {
      performers
    }

    ...on Concert {
      performingBand1
    }

    ...on Conference {
      speakers
      workshops
    }
  }
}
```

This query yields a single list of results, and the server could sort the events by start data by default:

```graphql
{
  "data": {
    "findEventsAtVenue": [
      {
        "id": "Festival-2",
        "name: "Festival 2",
        "minAgeRestriction": 21,
        "startsAt": "2018-10-05T14:48:00.000Z",
        "performers": [
          "The Singers",
          "The Screamers"
        ]
      },
      {
        "id": "Concert-3",
        "name": "Concert 3",
        "minAgeRestriction": 18,
        "startsAt": "2018-10-07T14:48:00.000Z",
        "performingBand": "The Jumpers"
      },
      {
        "id": "Conference-4",
        "name": "Conference 4",
        "minAgeRestriction": null,
        "startsAt": "2018-10-09T14:48:00.000Z",
        "speakers": [
          "The Storytellers"
        ],
        "workshops": [
          "Writing",
          "Reading"
        ]
      }
    ]
  }
}
```

Results are returned as a single collection of events. Using interfaces to represent common characteristics is very helpful for sorting results.

## Unions

GraphQL's type system also features Unions. Unions are identical to interfaces, except that they don't define a common set of fields. Unions are generally preferred over interfaces when the possibly types do not share a logical hierarchy.

A search result might represent many different types. Using the *Event* schema, we could define a *SearchResult* union as follows:

```graphql
type Query {
  # Retrieve Events at a specific Venue
  findEventsAtVenue(venueId: ID!): [Event]
  # Search across all content
  search(query: String!): [SearchResult]
}
union SearchResult = Conference | Festival | Concert | Venue
```

In this case, to query any field on our *SearchResult* union, we must use fragments:

```graphql
query {
  search(query: "Madison") {
    ...on Venue {
      id
      name
      address
    }

    ...on Festival {
      id
      name
      performers
    }

    ...on Concert {
      id
      name
      performingBand
    }

    ...on Conference {
      speakers
      workshops
    }
  }
}
```

## Type Resolution in AWS AppSync

Type resolution is the mechanism by which the GraphQL engine identifies a resolved value as a specific object type.

Coming back to the union search example, provided our query yielded results, each item in the results list must present itself as one of the possible types that the *SearchResult* union defined (*Conference*, *Festival*, *Concert*, or *Venue*).

Because the logic to identify a *Festival* from a *Venue* or a *Conference* is dependent on the application requirements, the GraphQL engine must be given a hint to identify our possible types from the raw results.

With AWS AppSync, this hint is represented by a meta field named __typename, whose value corresponds to the identified object type name. __typename is required for return types that are interfaces or unions.

## Type Resolution Example

```graphql
schema {
  query: Query
}

type Query {
  # Retrieve Events at a specific Venue
  findEventsAtVenue(venueId: ID!): [Event]
  # Search across all content
  search(query: String!): [SearchResult]
}

union SearchResult = Conference | Festival | Concert | Venue

type Venue {
  id: ID!
  name: String!
  adress: String
  maxOccupancy: Int
}

interface Event {
  id: ID!
  name: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
}

type Festival implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  performers: [String]
}

type Conference implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  speakers: [String]
  workshops: [String]
}

type Concert implements Event {
  id: ID!
  name: String!
  startsAt: String
  endsAt: String
  venue: Venue
  minAgeRestriction: Int
  performingBand: String
}
```

Let's attach a resolver to the *Query.search* field. In the console, we can choose **Attach Resolver**, create a new **Data Source** of type *NONE*, and then name it *StubDataSource*. For the sake of this example, we can pretend that we fetched results from an external source, and hard code the fetched results in the request mapping template.

```json
{
  "version": "2017-02-28",
  "payload":
  // We are effectively mocking our search results for this example
  [
    {
      "id": "Venue-1",
      "name": "Venue 1",
      "address": "2121 7th Ave, Seattle, WA 98121",
      "maxOccupancy": 1000
    },
    {
      "id": "Festival-2",
      "name": "Festival 2",
      "performers": ["The Singers", "The Screamers"],
    },
    {
      "id": "Concert-3",
      "name": "Concert 3",
      "performingBand": "The Jumpers"
    },
    {
      "id": "Conference-4",
      "name": "Conference 4",
      "speakers": ["The Storytellers"],
      "workshops": ["Writing", "Reading"]
    }
  ]
}
```

In the application, we chose to return the type name as part of the *id* field. The type resolution logic only consists of parsing the *id* field to extract the type name and adding the __typename field to each of the results. We can perform the logic in the response mapping template as follows:

**Note:** We can also perform this task as part of our Lambda function, if we are using the Lambda data store.

```javascript
// foreach ($result in $context.result)
  // Extract type name form the id field.
  // set($typeName = $result.id.split("-")[0])
  // set($ignore = $result.put("__typename", $typeName))
// end
$util.toJson($context.result)
```

We can run the following query:

```graphql
query {
  search(query: "Madison") {
    ...on Venue {
      id
      name
      address
    }

    ...on Festival {
      id
      name
      performers
    }

    ... on Concert {
      id
      name
      performingBand
    }

    ...on Conference {
      speakers
      workshops
    }
  }
}
```

The query yields the following results:

```graphql
{
  "data": {
    "search": [
      {
        "id": "Venue-1",
        "name": "Venue 1",
        "address": "2121 7th Ave, Seattle, WA 98121"
      },
      {
        "id": "Festival-2",
        "name": "Festival 2",
        "performers": [
          "The Singers",
          "The Screamers"
        ]
      },
      {
        "id": "Concert-3",
        "name": "Concert 3",
        "performingBand": "The Jumpers"
      },
      {
        "speakers" [
          "The Storytellers"
        ],
        "workshops": [
          "Writing",
          "Reading"
        ]
      }
    ]
  }
}
```