# AWS AppSync SDK

## Configuration

The AWS SDKs support configuration through a centralized file called *aws-exports.js* that defines our AWS regions and service endpoints. We obtain this file in one of two ways, depending on whether we are creating our AppSync API in the AppSync console or using the Amplify CLI

- If we are creating our API in the console, the *aws-exports.js* file we download is already populated for our specific API. We want to place the file in the *src* directory of our JavaScript project.

- If we are creating our API with the Amplify CLI (using *amplify add api*), the *aws-exports.js* file is automatically downloaded and updated every time we run *amplify push* to update our cloud resources. The file is placed in the *src* directory that we choose when setting up our JavaScript project.

## Code Generation

To execute GraphQL operations in JavaScript, we need to have GraphQL statements (for example, queries, mutations, or subscriptions) to send over the network to the server. We can optionally run a code generation process to do this for use. The Amplify CLI toolchain makes this easy by automatically pulling down our schema and generating default GraphQL queries, mutations, and subscriptions. If our client requirements change, we can alter these GraphQL statements and our JS project will automatically pick them up. We can also generate TypeScript definitions with the CLI and regenerate our types.

## AppSync APIs Created in the Console

After installing the Amplify CLI, we can open up a terminal, go to our JavaScript project root, and then run the following:

```javascript
amplify init
amplify add codegen --apiId XXXXXX
```

The **XXXXXX** is the unique AppSync API identifier that we can find in the console in the root of our API's integration page. When we run this command, we can accept the defaults, which create a *./src/graphql* folder with our statements.

## AppSync APIs Created Using the CLI

Navigate in our terminal to a JS project directory and run the following:

```javascript
$amplify init
$amplify add api
```

Select *GraphQL* when prompted for service type.

The **add api** flow will ask us some questions, such as if we already have an annotated GraphQL schema, etc.

AWS AppSync API keys expire seven days after creation, and using API KEY authentication is only suggested for development. To change AWS AppSync authorization type after the initial configuration, use the **$ amplify update api** command and select **GraphQL**.

When we update our backend with the *push* command, we can go to the *AWS AppSync Console* and see that a new API is added under *APIs* menu item:

```javascript
$ amplify push
```

The **amplify push** process will prompt us to enter the codegen process and walk through configuration options. Accept the defaults and it will create a *./src/graphql* folder structure with our documents. We also will have an aws-exports.js file that the AppSync client will use for initialization. At any time we can open the AWS console for our new API directly by running the following command:

```javascript
amplify console api
> GraphQL
```

This will open the AWS AppSync console for us to run Queries, Mutations, or Subscriptions at the server and see the changes in the client app.

## Dependencies

To use AppSync in our JS project, we need to add the following dependencies:

```javascript
yarn add aws-appsync graphql-tag
```

## Client Initialization

In our app's entry point, import the AWS AppSync Client and instantiate it:

```javascript
import gql from "graphql-tag"
import AWSAppSyncClient, { AUTH_TYPE } from "aws-appsync";
import awsconfig from "./aws-exports"

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY, 
    // or type: awsconfig.aws_appsync_authenticationType,
    apiKey: awsconfig.aws_appsync_apiKey,
  }
})
```

## Run a Query

Now that the client is configured, we can run a GraphQL query. The syntax is **client.query({query: QUERY})** which returns a **Promise** we can optionally **await** on. The **QUERY** is a GraphQL document that we can write ourselves or use the statements which **amplify codegen** created automatically. For example, if we have a **ListTodos** query, our code will look like the following:

```javascript
import {listTodos} from "./graphql/queries";

client.query({
  query: gql(listTodos)
}).then(({data: {listTodos}}) => {
  console.log(listTodos.items)
})
})
```

If we want to change the **fetchPolicy** to something like **cache-only** and not retrieve data over the network, we need to wait for the cache to be hydrated (instantiate an in-memory object from storage for the Apollo cache to use).


```javascript
import {listTodos} from "./graphql/queries";

(async () => {
  await client.hydrated();

  var result = await client.query({
    query: gql(listTodos),
    fetchPolicy: 'cache-only',
  });
  console.log(result.data.listTodos.items)
})()
```

It is recommended to leave the default of **fetchPolicy: 'cache-and-network'** for usage with AppSync.

## Run a Mutation

To add data, we need to run a GraphQL mutation. The syntax is **client.mutate({mutation:MUTATION, variables: vars})**, which like like a query returns a *Promise*. The **MUTATION** is a GraphQL document we can write ourself our use the statements which **amplify codegen** created automatically. **variables** are an optional object if the mutation requires arguments. For example, if we have a **createTodo** mutation, our code will look like the following:

```javascript
import {createTodo} from './graphql/mutations';

(async () => {
  var result = await client.mutate({
    mutation: gql(createTodo),
    variables: {
      input: {
        name: 'Use AppSync',
        description: 'Realtime and Offline',
      }
    }
  });
  console.log(result.data.createTodo);
})();
```

## Subscribe to Data

Finally, it's time to set up a subscription to real-time. The syntax is **client.subscribe({ query: SUBSCRIPTION })** which returns an **Observable** that we can subscribe to with **.subscribe()** as well as **.unsubscribe()** when the data is no longer necessary in our application. For example, if we have a **onCreateTodo** subscription, our code might look like the following:

```javascript
import { onCreateTodo } from "./graphql/subscriptions";

let subscription;

(async () => {
  subscription = client.subscribe({query: gql(onCreateTodo)}).subscribe({
    next: data => {
      console.log(data.data.onCreateTodo);
    },
    error: error => {
      console.warn(error);
    }
  })
})();

// Unsubscribe after 10 secs
setTimeout(() => {
  subscription.unsubscribe();
  }, 100000);
```

Note that since **client.subscribe** returns an **Observable**, we can use *filter*, *map*, *forEach* and other stream related functions. When we subscribe, we'll get back a subscription object that we can use to unsubscribe.

Subscriptions can also take input types like mutations, in which case they will be subscribing to particular events based on the input.

# Client Architecture

The AppSync client supports offline scenarios with a programming model that provides a 'write through cache'. This allows us to both render data from the UI when offline as well as add/update through an "optimistic response".

Our application code will interact with the AppSync client to perform GraphQL queries, mutations, or subscriptions. The AppSync client automatically performs the correct authorization methods when interfacing with the HTTP layer adding API Keys, tokens, or signing requests depending on how we have configured our setup. When we do a mutation, such as adding a new item (like a blog post) in our app, the AppSync client adds this to a local queue (persisted to disk with Local Storage, Async Storage, or other mediums depending on our JavaScript platform configuration) when the app is offline. When network connectivity is restored, the mutations are sent to AppSync in serial, allowing us to process the reponses one by one.

Any data returned by a query is automatically written to the Apollo Cache (e.g. "Store") that is persisted to the configured medium. The cache is structured as a key value store using a reference structure. There is a base "Root Query" where each subsequent query query resides and then references their individual item results. We specify the reference key (normally "id") in our application code. An example of the cache that has stored results from a "listPosts" query and "getPost(id:1)" query:

Key: ROOT_QUERY
Value: [ROOT_QUERY.listPosts, ROOT_QUERY.getPost(id: 1)]

Key: ROOT_QUERY.listPosts
Value: {0,1,...,N}

Key: Post:0
Value: {author: "Nadia", content: "ABC"}

Key: Post: 1
Value: {author: "Shaggy", content: "DEF"}

...

Key: Post:N
Value: {author: "Pancho", content: "XYZ"}

Key: ROOT_QUERY.getPost(id:1)
Value: ref: $Post:1

Notice that the cache keys are normalized where the *getPost(id:1)* query references the same element that is part of the *listPosts* query. This happens automatically in JavaScript applications by using *id* as a common cache key to uniquely identify the objects. We can choose to change the cache key with the *cacheOptions: {dataIdFromObject}* method creating the *AWSAppSyncClient*:

```javascript
const client = new AWSAppSyncClient({
  url: awsconfig.aws_app_sync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey,
  },
  cacheOptions: {
    dataIdFromObject: (obj) => `${obj.__typename}: ${obj.myKey}`
  }
})
```

If we are performing a mutation, we can write an "optimistic response" anytime to this cache even if we are offline. We use the AppSync client to connect by passing in the query to update, reading the items off the cache. This normally returns a single item or a list of items, depending on the GraphQL response type of the query to update. At this point, we would add to the list, remove, or update it as appropriate and write back the response to the store, persisting it to disk. When we reconnect to the network, any responses from the service will overwrite the changes as the authoritative response.

## Configuration Options

**disableOffline**: If we don't want/need offline capabilities, this option skips the creation of a local store to persist the cache and mutations made while offline:

```javascript
var client = new AppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey,
  },
  disableOffline: true
})
```

**conflictResolver**: When clients make a mutation, either online or offline, they can send a version number with the payload (named *expectedVersion*) for AWS AppSync to check before writing to Amazon DynamoDB. A DynamoDB resolver mapping template can be configured to perform conflict resolution in the cloud.

[Resolver Mapping Template Reference](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference-dynamodb.html#aws-appsync-resolver-mapping-template-reference-dynamodb-condition-expressions)

If the service determines it needs to reject the mutation, data is sent to the client and we can optionally run a "custom conflict resolvers" to perform client-side conflict resolution.

- **mutation**: GraphQL statement of a mutation
- **mutationName**: Optional if a name of a mutation is set on a GraphQL statement
- **variables**: Input parameters of the mutation
- **data**: Response from AWS AppSync of actual data in DynamoDB
- **retries**: Number of times a mutation has been retried

An example of passing a *conflictResolver* to the *AWSAppSyncClient* object:

```javascript
var conflictResolver = ({mutation, mutationName, variables, data, retries}) => {
  switch (mutationName) {
    case 'UpdatePostMutation':
      return {
        ...variables,
        expected: data.version
      };
    default:
      return false;
  }
}

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey,
  },
  conflictResolver: conflictResolver
})
```

In the example above, we could do a logical check on the mutationName. If we return an object with variables for the mutation, this will automatically rerun the mutation with the correct version that AWS AppSync returned.

**Note:** This is not a recommended pracetice. Usually it is best to let AppSync define conflict resolution to prevent race conditions from occuring. If we don't want to retry, simple return "DISCARD".

## Offline Configuration

When using the AWS AppSync SDK offline capabilities (e.g. **disableOffline: false**), we can provide configurations in the **offlineConfig** key:

- Error Handling: (*callback*)
- Custom Storage Engine: (*storage*)
- A Key Prefix for the Underlying Store (*keyPrefix*)

### Error Handling

If a mutation is done while the app was offline, it gets persisted to the platform storage engine. When coming back online, it is sent to the GraphQL Endpoint. When a response is returned by the API, the SDK will notify us of the success or error using the callback provided in the **offlineConfig** parameter as follows:

```javascript
var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey,
  },
  offlineConfig : {
    callback: (err, succ) => {
      if(err) {
        var {mutation, variables} = err;

        console.warn(`ERROR for ${mutation}`, err);
      } else {
        var {mutation, variables} = succ;

        console.info(`SUCCESS for ${mutation}`, succ)
      }
    }
  }
})
```

**NOTE:** If the app was closed and we re-opened it and there were errors, this would be represented in the error callback. However, if we're doing a mutation and the app is still online and the server rejects the write, we will need to handle it with a standard **try/catch**:

```javascript
(async () => {
  var variables = {
    input: {
      name: "Use AppSync",
      description: "Realtime and Offline",
    }
  };

  try {
    var result = await client.mutate({
      mutation: gql(createTodo),
      variables: variables
    });
  } catch(err) {
    console.warn('Error sending mutation: ', err);
    console.warn(variables);
  }
})()
```

**NOTE:** The SDK will automatically retry for standard network errors, however access errors or other unrelated errors we will need to handle them ourselves.

### Custom Storage Engine

We can use any custom storage engine from the redux-persist supported engines list:

[Redux-Persist Supported Engines](https://github.com/rt2zz/redux-persist#storage-engines)

```javascript
import * as localForage from "localforage";

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey,
  },
  offlineConfig: {
    storage: localForage
  }
})
```

### Key Prefix

The AWSAppSyncClient persists its cache data to support offline scenarios. Keys in the persisted cache will be prefixed by the provided *keyPrefix*

This prefix is required when offline support is enabled and we want to use more than one client in our app:

[AppSync Multi Auth](https://aws-amplify.github.io/docs/js/api#aws-appsync-multi-auth)

## Offline Mutations

All query results are automatically persisted to disk with the AppSync client. For updating through mutations when offline, we will need to use an "optimistic response" by writing directly to the store. This is done by querying the store directly with *cache.readQuery({query:someQuery})* to pull the records for a specific query that we wish to update. We can do this manually with *update* functions or use the *buildMutation* and *buildSubscription* built-in helpers that are part of the AppSync SDK (we strongly recommend using these helpers).

[Offline Helpers Docs](https://github.com/awslabs/aws-mobile-appsync-sdk-js/blob/master/OFFLINE_HELPERS.md)

The *update* functions can be called two or more times when using an *optimisticResponse* depending on the number of mutations we have in our offline queue **and** if we are currently offline. This is because the Apollo client will call the function once for the local optimistic write and a second time for the server response.

[Apollo Cache Updates Docs](https://www.freecodecamp.org/news/how-to-update-the-apollo-clients-cache-after-a-mutation-79a0df79b840/)

The AppSync client when offline will automatically resolve the promise for the server response and place our mutation in a queue for later processing. When we come back online, each item in the queue will be executed in serial and its corresponding *update* function will run as well as triggering the optimistic update of the pending items in the queue. This is to ensure that the cache is consistent when rendering the UI. This means that our *update* functions should be idempotent.

The below code shouws how we would update the **CreateTodoMutation** mutation from earlier by creating a **optimisticWrite(CreateTodoInput, createTodoInput)** helper method that has the same input. This adds an item to the cache by first adding query results to a local array with *items.addAll(response.data().listTodos().items())* followed by the individual update using *items.add()*. We commit the record with *client.getStore().write()*. This example uses a locally generated unique identifier which might be enough for our app, however if the AppSync response returns a different value for *ID* (**which many times is the case as best practice is generation of IDs at the service layer**) then we will need to replace the value locally when a response is received. This can be done in the *onResponse()* method of the top level mutation callback by again querying the store, removing the item and calling *client.getStore().write().*

#### With Helper

An example of using the *buildMutation* helper to add an item to the cache:

```javascript
import {buildMutation} from 'aws-appsync'
import {listTodos} from './graphql/queries'
import {createTodos, CreateTodoInput} from './graphql/mutations'

(async () => {
  var result = await client.mutate(buildMutation(client, 
    gql(createTodo), {
      inputType: gql(CreateTodoInput),
      variables: {
        input: {
          name: 'Use AppSync',
          description: 'Realtime and Offline'
        }
      }
    },
    (_variables) => [gql(listTodos)], 
    'Todo'
  ))
  console.log(result)
})
```

### Without Helper

An example of writing an *update* function manually to add an item to the cache:

```javascript
import {v4 as uuid} from 'uuid';
import {listTodos} from './graphql/queries';
import {createTodo} from './graphql/mutations';

(async () => {
  var result = await client.mutate({
    mutation: gql(createTodo),
    variables: {
      input: {
        name: 'Use AppSync',
        description: 'Realtime and Offline'
      }
    },
    optimisticResponse: () => ({
      createTodo: {
        // This type must match the return type of the query below
        __typename: 'Todo',
        id: uuid(),
        name: 'Use AppSync',
        description: 'Realtime and Offline',
      }
    }),
    update: (cache, {data: {createTodo}}) => {
      var query = gql(listTodos);

      // Read query from cache
      var data = cache.readQuery({query});

      // Add a newly created item to the cache copy
      data.listTodos.items = [
        ...data.listTodos.items.filter(item => item.id !== createTodo.id),
        createTodo
      ];

      // Overwrite the cache with the new results
      cache.writeQuery({query, data});
    }
  })

  console.warn(result);
})()
```

We might add similar code in our app for updating or deleting items using an optimistic response, it would look largely similar except that we might overwrite or remove an element from the **data.listTodos.items** array.

## Authentication Modes

For client authorization, AppSync supports API Keys, Amazon IAM credentials, Amazon Cognito User Pools, and 3rd party OIDC providers. This is inferred from the **aws-exports.js** when we call *.awsConfiguration()* on the *AWSAppSyncClient* builder.

### API Key Auth

API Key is the easiest way to set up and prototype our application with AppSync. It's also a good option if our application is completely public. If our application needs to interact with other AWS services besides AppSync, such as S3, we will need to use IAM credentials provided by Cognito Identity Pools, which also supports "Guest" access. For manual configuration:

```javascript
var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.API_KEY,
    apiKey: awsconfig.aws_appsync_apiKey
  }
});
```

### Cognito User Pools Auth

Amazon Cognito User Pools is the most common service to use with AppSync when adding use Sign-Up and Sign-In to our application. If our application needs to interact with other AWS services besides AppSync, such as S3, we will need to use IAM credentials with Cognito Identity Pools. The Amplify CLI can automatically configure this for us when running **amplify add auth** and can also automatically federate User Pools with Identity Pools. This allows us to have both User Pool credentials for AppSync and AWS credentials for S3. We can then use the *Auth* category for automatic credentials refresh.

### Manual Configuration

```javascript
import Amplify, {Auth} from 'aws-amplify';
import awsconfig from './aws-exports';

Amplify.configure(awsconfig);

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.AMAZON_COGNITO_USER_POOLS,
    jwtToken: async () => (await Auth.currentSession()).getIdToken().getJwtToken(),
  }
});
```

### IAM Auth

When using AWS IAM in a mobile application we should leverage Amazon Cognito Identity Pools. The Amplify CLI will automatically configure this for us when running amplify add auth. We can then use the *Auth* category for automatic credentials refresh. For manual configuration, add the following snippet:

```javascript
import Amplify, {Auth} from 'aws-amplify';
import awsconfig from './aws-exports'

Amplify.configure(awsconfig);

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.AWS_IAM,
    credentials: () => Auth.currentCredentials(),
  },
});
```

### OIDC

If we are using a 3rd party OIDC provider, we will need to configure it and manage the details of token refreshes yourself. Update the **aws-exports.js** file:

```javascript
import Amplify, {Auth} from 'aws-amplify';
import awsconfig from './aws-exports';

Amplify.configure(awsconfig);

// Should be an async function that handles token refresh
var getOIDCToken = async () => await 'token';

var client = new AWSAppSyncClient({
  url: awsconfig.aws_appsync_graphqlEndpoint,
  region: awsconfig.aws_appsync_region,
  auth: {
    type: AUTH_TYPE.OPENID_CONNECT,
    jwtToken: () => getOIDCToken(),
  },
});
```

## Complex Objects
