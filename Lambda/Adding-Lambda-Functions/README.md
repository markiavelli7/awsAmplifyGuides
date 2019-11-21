# Build a Lambda Function

```javascript
amplify add function
```

This is the default hello-world lambda function

```javascript
exports.handler = function (event, context) { //eslint-disable-line
  console.log(`value1 = ${event.key1}`);
  console.log(`value2 = ${event.key2}`);
  console.log(`value3 = ${event.key3}`);
  context.done(null, 'Hello World'); // SUCCESS with message
};
```

A tax calculator function:

```javascript
exports.handler = function(event,context) {
  var taxTable = {
    NY: 8.49, CA: 8.56, OK: 8.92, FL: 7.05,
    IL: 8.74, PA: 6.34, WA: 9.17, TX: 8.19
  }

  var price
  var taxRate
  if (event.arguments) {
    // graphql request
    price = event.arguments.price
    taxRate = tabTable[event.arguments.location]
  } else {
    price = event.price
    taxRate = taxTable[event.location]
  }

  var calculatedTax = (taxRate / 100) * price
  calculatedTax = Math.round((calculatedTax + Number.EPSILON) * 100) / 100

  var finalPrice = (calculatedTax + price).toFixed(2)

  var response = {
    calculatedTax,
    finalPrice
  }

  context.done(null, response)
}
```

# Mocking the Function

To run the function that we just created, we use the command amplify mock function <function name>

If we don't know the name of the function we can run:

```javascript
amplify status
```

to get the name of all of the amplify resources in our project, including the name of the function that we just created.

```javascript
amplify mock function calculateTax
```

Inside of the function folder, we are provided with an event.json file. This is what our Lambda function will use as the event, we can customize the data that we are sending in to our function:

```javascript
event.json

{
  "price": 100.00,
  "location": "WA"
}
```

