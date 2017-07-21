# Local Emulator

> Emulate your Serverless functions locally.

---

1. [Getting started](#getting-started)
2. [Development](#development)
3. [Functionality](#functionality)
  - 3.1 [General](#general)
  - 3.2 [Deployment](#deployment)
  - 3.3 [Invocation](#invocation)
  - 3.4 [Middlewares](#middlewares)
4. [APIs](#apis)
  - 4.1 [HTTP API](#http-api)
    + 4.1.1 [Deploy function](#deploy-function)
    + 4.1.2 [Invoke function](#invoke-function)

---

## Getting started

1. Clone the repository
2. Run `npm install` to install all dependencies
3. Run `npm build` to build the project (the build artifacts can be found in `dist`)
4. Run `npm start` to start the emulator

## Development

You can run `npm watch` to automatically re-build the files in the `src` directory when they change.

Additionally you can use Docker to run and develop everything inside a container.

Spinning up the Docker container is as easy as `docker-compose run -p 8080:8080 node bash`.

## Functionality

### General

The Local Emulator is a software which makes it possible to emulate cloud provider specific functionality on your local machine in an offline-first fashion.

The Local Emulator can be used to deploy and invoke serverless functions w/o the need to go through the tedious process of setting up your cloud provider account or deploying your functions to the cloud infrastructure.

This enables new ways of doing serverless development offline first while introducing way faster feedback cycles.

Technically speaking the Local Emulator is a long running process (Daemon) which exposes an API and accepts different calls to the different API endpoints in order to perform actions and control its behavior / functionality.

**Note:** The API is only used to control the Local Emulator. It is **NOT** the same as an e.g. AWS API Gateway or smth. similar.

### Deployment

On every function deployment the following happens behind the scenes:

- The .zip file will be extracted
- The content of the zip file will be moved into the following directory structure:

```
|__ storage
    |__ functions
        |__ <service-name>
            |__ <function-name>
                |__ code
                |__ function.json
```

The root directory is the `storage` directory. It's a place where all artifacts will be stored.

The `functions` directory is the place where all the function-related artifacts are stored.

A directory for the specific service is created and contains dedicated directories for each function.

The `code` directory is the place where the actual function code is stored (the unzipped content of the submitted `.zip` file).

The `function.json` file is created on the fly and contains important information about the function configuration (e.g. where the handler can be found) and its metadata.

The proposed directory structure makes it easy for the Local Emulator to follow a convention-over-configuration approach where artifacts are stored in a predictable way without having to introduce a local DB with state information about all the deployed functions and services.

(*However we might introduce such a feature later on when we decide to e.g. display some information about the state of the Local Emulator at `GET /`*).

### Invocation

When invoking a function the Local Emulator will simply look for the function directory (see above how the naming schema helps with the lookup), determines the runtime based on the file extension of the function handler which is found in `function.json` and starts the execution phase which will happen in a dedicated child process (more on that later).

The invocation data is extracted from the incoming requested and passed to a so-called `wrapper` script via `stdin`. The wrapper script is an implementation of language specific logic and functionality and is written in the target runtime language. It's responsible to setup the execution environment, require the function and pass the event payload to the function (you can think of it as a language specific container). Furthermore it will marshall the returned data and pass it back to the Local Emulators parent process which will then transform it into a JSON format and sends it back via a HTTP response.

The whole invocation happens in a `child_process.spawn()` call to ensure that the Local Emulator won't crash when a function misbehaves.

This abstraction layer makes it easy to introduce other runtimes later on. Furthermore the way the event data is handled by the Local Emulator is always the same and independent of the function handler signature since the wrapper encapsulates the logic to marshall and unmarshall the data which is handed over to the function.

Here's a sample call to a wrapper script which is written in Node.js. It will require the function and pass the JSON data as event data to the required function.

```bash
echo '{ "service": "my-service", "function": "my-function", "payload": { "event": {}, "context": {}, "callback": () => {} } }' | node node.js
```

Here's another example which does essentially the same for a Python environment:

```bash
echo '{ "service": "my-service", "function": "my-function", "payload": { "event": {}, "context": {}, "callback": () => {} } }' | python python.py
```

### Middlewares

The Local Emulator provides a middleware concept which makes it possible to use own, custom code to modify the way the Local Emulator handles and passes the data to the functions and back from the functions to the invoker.

Middlewares can be implemented against different lifecycle events. Right now the lifecycle events are:

| Lifecycle | Description |
| --- | --- |
| `preInvoke` | Right after, but before the function which invokes the specific function is executed |
| `postInvoke` | After the function is invoked, but before it's result is passed back via the API |

The Local Emulator uses own `core-middlewares` to e.g. modify the data and prepare it so that it can be passed in the provider specific handler format.

Here's an example of a `my-middleware.js` file which shows how middlewares are implemented:

```javascript
const preInvoke = data => Promise.resolve(data);
const postInvoke = data => Promise.resolve(data);

module.exports = { preInvoke, postInvoke };
```

Middlewares are loaded and executed in the following order:

1. Load / Execute the core middlewares (according to the `core-middlewares` directory)
1. Load / Execute the custom user defined middlewares

## APIs

The local emulator exposes different APIs which makes it possible to interact with it and perform specific actions.

Examples for such actions could e.g. be the deployment or invocation of functions.

Right now the local emulator only exposes a HTTP API. However other API types such as (g)RPC are imaginable.

### HTTP API

The local emulator exposes a HTTP API which makes it possible for other services to interact with it via HTTP calls.

#### Deploy function

`POST /v0/emulator/api/functions`

Request:

- `functionName` - `string` - **required** The name of the function
- `serviceName` - `string` - **required** The service the function belongs to
- `config` - `object`: - **required** Additional function configuration
  + `handler` - `string` - **require** The exported function which should be used
- `data` - `buffer` - **required** The zip file data which contains the functions code

Response:

- `functionName` - `string` - The name of the function
- `serviceName` - `string` - The service the function belongs to
- `config` - `object` - Additional function configuration

#### Invoke function

`POST /v0/emulator/api/functions/invoke`

Request:

- `functionName` - `string` - **required** The name of the function
- `serviceName` - `string` - **required** The service the function belongs to
- `payload` - `object` - **required** The event payload the function should receive

Response:

- `functionName` - `string` - The name of the function
- `serviceName` - `string` - The service the function belongs to
- `payload` - `object` - The event payload the function should receive
