# Automatically Add a User to A Cognito User Pool With a Lambda Trigger

AWS Amplify Transform provides an @auth directive. With it, we have the ability to grant access to certain parts of our API based on eithre static or dynamic groups.

Say we have a task api and we want:

1. Each user to login to use our app and greant read access to our API
2. To have managers, that can additionally create and update a task
3. To have admins, that can do everything

### Upon Sign Up, we expect all users to be a group named 'Users'. Managers and Admins will be added to the groups by hand in the AWS Management Console, but each user will at least belong to the 'Users' group.

## Update Auth from the Amplify Console
```javascript
ampliy update auth
```

From there, select the 'Create Pools' option and enter in the required user groups.

## Add a User to a Pool With a Lambda Trigger

Cognito offers triggers during certain life-cycle events. One of these triggers is 'PostConfirmation', which may run a Lambda function after a user was successfully confirmed.

### Add a New Lambda Function

The AWS Amplify CLI offers the ability to add a function to our project:

```javascript
amplify add function
```

We will choose the name 'addUserToGroup', when asked for the friendly name and Lambda function name, and as a template, we want Hello world function.

Now we want add our function to the new index.js file:

```javascript
exports.handler = function afterConfirmationTrigger(event, context) {
  //eslint-disable-line
  var AWS = require("aws-sdk")
  var cognitoISP = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  })

  var params = {
    GroupName: "user",
    UserPoolId: event.userPoolId,
    Username: event.userName,
  }

  cognitoISP
    .adminAddUserToGroup(params)
    .promise()
    .then(() => context.done(null, event))
    .catch(err => context.done(err, event))
}
```

In the event.json file, we can change the payload content to our liking to test the function.

```javascript
{
  "userPoolId": "the-user-pool-id",
  "userName": "user1",
  "userPoolGroupName": "Users"
}
```

We can then test our function:

```javascript
amplify invoke function addUserToGroup
```

However, our function will not work yet. We still need to give our function permission to do its work.