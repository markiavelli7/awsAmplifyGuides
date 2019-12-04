# Setting Up AppSync With React

## Add the Amplify Backend

```bash
amplify init
```
**Enter a name for the project** project-name
**Enter a name for the environment** dev
**Choose your default editor** Visual Studio Code
**Choose the type of app that you're building** javascript
**Please tell us about your project** react
**Source Directory path** src
**Distribution Directory Path** public-**GATSBY** build-**React**
**Build Command** npm run-script build
**Start Command** npm run-script start
**Do you want to use an AWS profile (Y/n)** Y
****

## Add the Amplify API

```bash
amplify add api
```
**Please select from one of the below mentioned services** GraphQL
**Provide API name** api-name
**Choose an authorization type for the API** API key
For testing, we just want to use the API key option
**Do you have an annotated GraphQL schema? (y/N)**
**Do you want a guided schema creation?** Yes
**What best describes your project** Single object with fields
What we enter in here doesn't really matter, we are going to create our own schema anyway.
**Do you want to edit the schema now?**

```graphql
type Venue @model {
  id: ID!
  venueName: String!
  createdAt: String
  reviews: [Review] @connection(name: "VenueReviews")
}

enum ExperienceRating {
  GOOD
  FAIR
  POOR
}

type Review @model {
  id: ID!
  approved: Boolean!
  reviewerId: String!
  reviewerName: String!
  venueExperience: ExperienceRating!
  reviewTitle: String!
  reviewBody: String!
  reviewDate: String
  createdAt: String
  venue: Venue @connection(name: "VenueReviews")
  upvotes: [Upvote] @connection(name: "ReviewUpvotes")
  downvotes: [Downvote] @connection(name: "ReviewDownvotes")
}

type Upvote @model {
  id: ID!
  numberUpvotes: Int!
  upvoteOwnerId: String!
  upvote: Boolean!
  createdAt: String
  review: Review @connection(name: "ReviewUpvotes")
}

type Downvote @model {
  id: ID!
  numberDownvotes: Int!
  downvoteOwnerId: String!
  downvote: Boolean!
  createdAt: String
  review: Review @connection(name: "ReviewDownvotes")
}
```

## Push the Schema to the Cloud

```bash
amplify push
```

#### Opening the AppSync Console from the Command Line

```bash
amplify console api
```

From here we can run queries, look at our schema, etc.

## Adding Amplify Modules and Configuring Our Frontend App

1. Install the Dependencies

```bash
yarn add aws-amplify aws-amplify-react
```

2. Update the project to use Amplify

**Gatsby**

*gatsby-browser.js*

```javascript
import Amplify from "aws-amplify"
import awsconfig from "./src/aws-exports"

Amplify.configure(awsconfig)
```

**React**

*index.js*

```javascript
import Amplify from "aws-amplify"
import awsconfig from "./src/aws-exports"

Amplify.configure(awsconfig)
```

## Logging Data From AWS AppSync

In order to run queries, we need to import the queries that Amplify created for us. These are located inside of *src/graphql/queries.js*

```javascript
import { useEffect } from "react"
import { listVenues } from "../graphql/queries"
import { API, graphqlOperation } from 'aws-amplify'

async function getVenues() {
  try {
    var result = await API.graphql(graphqlOperation(listVenues))
    console.log(`All Posts: `, JSON.stringify(result.data, null, 2))
  } catch (err) {
    console.error(`Error: ${err}`)
  }
}

export function VenuePage({ data }) {
  useEffect(() => {
    getVenues()
  }, [])
  ...
```

## Interacting with the AppSync Data

As the data comes in, we will need to store it somewhere, this is what we will use state for.

```javascript
import React, { useState, useEffect } from "react"
import { API, graphqlOperation } from "aws-amplify"
import { createVenue } from "../graphql/mutations"

function CreatePost() {
  var [venue, setVenue] = useState("")
  useEffect(() => {}, [])

  async function submitForm(e) {
    e.preventDefault()

    var input = {
      venueName: venue,
      createdAt: new Date().toISOString(),
    }

    try {
      await API.graphql(graphqlOperation(createVenue, { input }))
      setVenue("")
    } catch (err) {
      console.error(`GraphQL operation failed: ${err}`)
    }
  }

  function inputChange(e) {
    setVenue(e.target.value)
  }

  return (
    <form onSubmit={e => submitForm(e)}>
      <input
        onChange={e => {
          inputChange(e)
        }}
        name="venueName"
        value={venue}
        type="text"
        placeholder="Venue Name"
        required
      />

      <button type="submit">Submit</button>
    </form>
  )
}

export default CreatePost
```

## Posting Data From the App

```javascript
import React, { useState, useEffect } from "react"

function CreatePost() {
  var [venue, setVenue] = useState("")
  useEffect(() => {}, [])

  function submitForm(e) {
    e.preventDefault()

    var input = {
      venueName: venue,
      createdAt: new Date().toISOString()
    }
  }

  function inputChange(e) {
    setVenue(e.target.value)
  }


  return (
    <form onSubmit={e => submitForm(e)}>
      <input
        onChange={e => {
          inputChange(e)
        }}
        name="venueName"
        value={venue}
        type="text"
        placeholder="Venue Name"
        required
      />

      <button type="submit">Submit</button>
    </form>
  )
}

export default CreatePost
```

## Adding the onCreate Subscription and Refreshing UI with Posts Automatically

#### Subscriptions can be computationally expensive, so we need to make sure that we can remove the subscription listener when this page no longer needs it:

**venues.js**

```javascript

```