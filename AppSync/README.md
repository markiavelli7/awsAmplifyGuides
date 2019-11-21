# Using Amplify AppSync

Initialize a new Amplify project within the root directory of the project

```javascript
amplify init
```

This will walk through basic project initialization.

Environment name is the default name for our environment, when starting a project, dev or local is a good name.

Source Directory Path: src
Distribution Directory Path: public
Build Command: yarn build
Start Command yarn develop

*Use aws profile*

### Once our Amplify project is successfully created, Amplify will add some new items to our project:

  * An amplify folder at the root of the project
  * An aws-exports.js file to whatever we specified in the setup as the Source Directory Path


### aws-exports.js

This file is automatically populated when we make updates to amplify.

### amplify Folder

#### #current-cloud-backend

Inside of #current-cloud-backend, there is a file called amplify-meta.json. This file contains the currently synced backend that we have deployed to our account. *We wouldn't ever edit this file, it is just a representation of what is currently deployed.*

#### backend

This folder is where we will write some code. We can make modifications to our Amplify project and push the new changes up to our account.

## Adding an API

```javascript
amplify add api
```
Go through the wizard setup, choosing authorization type, API name, and GraphQL Schema options.

If we choose to edit the default schema, we will see something like this:

```graphql
type Todo @model {
  id: ID!
  name: String!
  description: String!
}
```

@model is an aws directive that builds out additional functionality for our GraphQL API. @model is a special directive that will build out all of the additional CRUD and list operations in our schema. It will also create a DynamoDB database for us with all of the resolvers.

*After we edit the schema, run an amplify push to add the changes to our project.*

## Add the Gatsby Source GraphQL Plugin

```javascript
yarn add gatsby-source-graphql
```

### Configure gatsby-source-graphql in the gatsby-config.js file

```javascript
module.exports = {
  plugins: [
    resolve: 'gatsby-source-graphql',
    options: {
      typeName: "SWAPI",
      fieldName: 'swapi',
      url: "https://api.ethuanteohuasn.com/otuhe"
      headers: {
        'x-api-key': "nuhteotuheonstuhas"
      }
    }
  ]
}
```

## Opening the AppSync Console

```javascript
amplify console api
```

### Adding Data

From the GraphQL Editor, we can add our characters to the database.

## Querying for Data in Gatsby

In Gatsby, we have two ways to query for data:

1. If we are using a page component, we can export a query from it. The requirement for this is that it has to be used as part of page generation because there is special treatment for those files.

2. useStaticQuery. Inside of the function, we make a call to useStaticQuery. useStaticQuery is a React Hook that allows us to pull in data.

#### The shortcoming of useStaticQuery: you can't use variables with static queries. That is why they are called static queries. This only works when we can hardcode all of the values.