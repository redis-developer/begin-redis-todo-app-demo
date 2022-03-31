A super simple Begin CRUD(Create Read Update Delete) app that exemplifies a basic todo app that uses one static html page and four API endpoints that connect to a Redis database.

# Prerequisites

1. Follow [this tutorial](https://developer.redis.com/create/aws/redis-on-aws) to sign up for a free Redis Enterprise Cloud account.

2. Create an `.env` file in the root of this project and add the content below with your server secret added

```sh
@testing
REDIS_URL=Your redis url

@staging
REDIS_URL=Your redis url

@production
REDIS_URL=Your redis url
```

## Deploy your own

[![Deploy to Begin](https://static.begin.com/deploy-to-begin.svg)](https://begin.com/apps/create?template=https://github.com/begin-examples/node-redis)

Deploy your own clone of this app to Begin!

## Local development

- Follow the [Redis quickstart](https://redis.io/topics/quickstart) to run the DB locally
- Start the local dev server: `npm start`


## How does it work?

### Project structure

Now that your app is live on staging and running locally, let’s take a quick look into how the project itself is structured so you’ll know your way around. Here are the key folders and files in the source tree of your new app:

```
.
├── public/
│   ├── index.html
│   └── index.js
├── src/
│   ├── http/
│   │    ├── get-todos/
│   │    ├── post-todos/
│   │    ├── post-todos-000id/
│   │    └── post-todos-delete/
│   └── shared/
│        └── redis-client.js
└── app.arc
```

### public/index.html & public/index.js

The `public/index.html` is the page served to the browser. This is where our JSON data will be appended to a DOM element of our choosing. public/index.js is where we will write our function that fetches the JSON data from get /todos and displays it in our HTML page.

Your app utilizes built-in small, fast, individually executing cloud functions that handle HTTP API requests and responses. (We call those HTTP functions, for short.)

The HTTP function that handles requests:

```
get /todos is found in src/http/get-todos/
post /todos is found in src/http/post-todos/
post /todos/:id is found in src/http/post-todos-000id/
post /todos/delete is found in src/http/post-todos/delete.
```

## Manipulate data stored in Redis from an HTTP Function.

### src/shared/redis-client.js

The redis-client.js file is where we will write common code that will be used in all of our HTTP functions. This code will automatically get copied into each HTTP function during the hydration phase of the install build step.

### app.arc

Your app.arc file is where you will provision new routes and functions.

Infrastructure-as-code is the practice of provisioning and maintaining cloud infrastructure using a declarative manifest file. It’s like package.json, except for cloud resources like API Gateway, Lambda, and DynamoDB (all of which Begin apps use).

By checking in your Begin app’s project manifest (app.arc) file with your code, you can ensure you have exactly the cloud resources your code depends on. This is crucial for ensuring reproducibility and improving iteration speed.

### Access Redis Enterprise Cloud from HTTP Functions

Let’s dig into how we connect to Redis Enterprise Cloud and manipulate data via HTTP Functions. Open src/http/get-todos/index.js:

In the first four lines of the function:

- We require the http middleware from @architect/functions.
- Include some helper functions from our redis-client.js package which are shared with all of our HTTP functions
- Setup our handler to call the async functions clientContext and read in that order.

```
// src/http/get-todos/index.js
const { http } = require('@architect/functions')
const { createConnectedClient, clientContext, clientClose } = require('@architect/shared/redis-client')

exports.handler = http.async(clientContext, read)
```

Over in src/shared/redis-client.js we will find the clientContext function.

```
// src/shared/redis-client.js
async function clientContext(req, context) {
 process.env.ARC_ENV === 'testing' ?
   context.callbackWaitsForEmptyEventLoop = true :
   context.callbackWaitsForEmptyEventLoop = false
}
```

This function may look weird, and that’s because it is. When we are running locally the architect sandbox spins up a new Node.js process to handle each incoming request. Once the request is fulfilled the process exits. When the function runs as an AWS Lambda we will have the option of keeping the connection to Redis Enterprise Cloud open to serve multiple requests.

Now let’s look at the read function from src/http/get-todos/index.js. The first line in the function is where we await the createConnectedClient helper function.

```
// src/http/get-todos/index.js
const client = await createConnectedClient()
```

Our createConnectedClient function will setup some event listeners that will log the connect, error and end events which are useful when debugging your application. Then the function will connect to your Redis Enterprise Cloud database and return a connected redis client.

```
// src/shared/redis-client.js
const redis = require('redis');

const client = redis.createClient({
  url: process.env.REDIS_URL
})

async function createConnectedClient() {
  client.on('connect', () => {
    console.log("connected to redis");
  })

  client.on("error", (error) => {
    console.error(error);
  });

  client.on("end", () => {
    console.log("dis-connected from redis");
  });

  await client.connect()
  return client;
}
```

### Aside

Our todo list items will be stored in the Redis database as a hash. Each todo item will have a unique key, and the value of that key is the JSON stringified version of the todo. For example:

```
{
  'tmhfqiVZq8Y7TFnaQHwOS': '{"text":"publish the blog post"}',
  '6lHfPRoOMvm8EgWqHOxsS': '{"completed":true,"text":"create screenshots"}',
  'EcrTh_FrpWp3sHgN3-3Lw': '{"completed":true,"text":"write the docs"}'
}
```

The client from the redis npm package provides access to all of the out-of-the-box Redis commands. Since we are going to store our todos as a hash in the database, we will use the following commands:

- hGetAll: returns all of our todo list items stored at our key
- hSet: creates or updates a single todo list item
- hDel: removes a todo list item

In order to fetch our todos from our collection we execute the following code:

```
// src/http/get-todos/index.js
let todos = await client.hGetAll('todos')
```

The next line in src/http/get-todos.js is:

```
// src/http/get-todos/index.js
await clientClose()
```

The redis npm package provides two ways to close connections to the database. quit gracefully closes the connection and awaits a response from Redis. disconnect immediately closes the connection. Since we want our CRUD actions to succeed we will use quit.

```
// src/shared/redis-client.js
async function clientClose() {
  client.quit()
}
```

Finally we’ll return the todos as the HTTP functions response

```
// src/http/get-todos/index.js
return {
  statusCode: 200,
  cacheControl: 'no-cache, no-store, must-revalidate, max-age=0, s-maxage=0',
  json: todos
}
```

Since we are using the arc.http.async helper function we can use some shortcuts when we return a payload from the function.

- statusCode sets the response to 200 OK
- cacheControl sets the cache-control header
- json set the content-type header to application/json; charset=utf8 while also JSON encoding the todos array for us.



## Reference

- [Building Functional Web Apps with Redis](https://blog.begin.com/posts/2022-03-24-building-functional-web-apps-with-redis)
- [Quickstart](https://docs.begin.com/en/guides/quickstart/) - basics on working locally, project structure, deploying, and accessing your Begin app
- [Creating new routes](https://docs.begin.com/en/functions/creating-new-functions) - basics on expanding the capabilities of your app

Head to [docs.begin.com](https://docs.begin.com/) to learn more!
