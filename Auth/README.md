# Adding Auth From the Amplify Console

Follow this guide for setting up Federated Sign-In with Facebook and Google

https://dev.to/dabit3/the-complete-guide-to-user-authentication-with-the-amplify-framework-2inh


## Setup Authentication in Gatsby

1. Install the necessary dependencies

```javascript
npm i aws-amplify aws-amplify-react
```

2. Import the configuration created by Amplify in src/aws-exports.js and pass it into the Amplify client.

##### gatsby-browser.js
```javascript
import Amplify, {Auth} from 'aws-amplify'
import awsConfig from './src/aws-exports'
Amplify.configure(awsConfig)
```

3. Set Up Client-Only Routes in Gatsby

##### gatsby-node.js
```javascript
exports.onCreatePage({page, actions}) => {
  var {createPage} = actions
  // page.matchPath is a special key that is used for matching pages on the client
  if (page.path.match(/^\/dashboard/)) {
    page.matchPath = "/dashboard/*"
    createPage(page)
  }
}
```

When we create pages, it'll match any page with the path /dashboard/*. This tells Gatsby that these routes should only be rendered on the client.. So /dashboard/events or /dashboard/notifications is goning to render back to the dashboard page.

4. From the dashboard page, test setup with AWS provided Auth helper

```javascript
import {Auth} from "aws-amplify"

export default function UserDashboard() {
  return (
    <Layout>
      <h1>This is your User Dashboard. Enjoy!</h1>
      <button onClick={() => Auth.federatedSignIn()}>Sign In</button>
    </Layout>
  )
}
```