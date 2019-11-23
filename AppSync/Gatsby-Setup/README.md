## GraphQL API Gatsby Setup

#### These steps assume that we have a Gatsby project initialized as well as an Amplify AppSync API ready to go.

1. Install the Dependencies

```javascript
yarn add gatsby-source-graphql
```

2. Configure the gatsby-source-graphql Plugin

**gatsby-config.js**

```javascript
...
plugins: [
  {
    resolve: "gatsby-source-graphql",
    options: {

    }
  }
]
```

3. Using the internal Gatsby graphql package to query for data

```javascript
export const query = graphql`
  query {
  office {
    listDepartments {
      items {
        name
        manager {
          id
          name
        }
      }
    }
  }
}
`
```