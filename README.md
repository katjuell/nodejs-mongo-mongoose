# Containerizing a Node.js Application for Development

### Introduction

If you are actively developing an application, using [Docker](https://www.docker.com/) can simplify your workflow and the process of deploying your application to production. Working with containers in development offers the following benefits:
- Environments are consistent, meaning that you can choose the languages and dependencies you want for your project without worrying about conflicts.
- Environments are isolated, making it easier to troubleshoot issues and onboard new team members. 
- Environments are portable, allowing you to package and share your code with others. 

This tutorial will show you how to set up a development environment for a [Node.js](https://nodejs.org/) application using Docker. You will create two containers — one for the Node application and another for the [MongoDB](https://www.mongodb.com/) database — with [Docker Compose](https://docs.docker.com/compose/). Because this application works with Node and MongoDB, our setup will do the following: 
- Synchronize the application code on the host with the code in the container to facilitate changes during development. 
- Ensure that changes to the application code work without a restart.
- Create a user and password-protected database for the application's data.
- Persist this data.

At the end of this tutorial, you will have a working shark information application running on Docker containers:

![Complete Shark Collection](https://assets.digitalocean.com/articles/node_docker_dev/persisted_data.png)

## Prerequisites

To follow this tutorial, you will need:
- A development server running Ubuntu 18.04, along with a non-root user with `sudo` privileges and an active firewall. For guidance on how to set these up, please see this [Initial Server Setup guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04). 
- Docker installed on your server, following Steps 1 and 2 of [How To Install and Use Docker on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04). 
- Docker Compose installed on your server, following Step 1 of [How To Install Docker Compose on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04).

## Step 1 — Cloning the Project and Modifying Dependencies

The first step in building this setup will be cloning the project code and modifying its [`package.json`](https://docs.npmjs.com/files/package.json) file, which includes the project's dependencies. We will add [`nodemon`](https://www.npmjs.com/package/nodemon) to the project's [`devDependencies`](https://docs.npmjs.com/files/package.json#devdependencies), specifying that we will be using it during development. Running the application with `nodemon` ensures that it will be automatically restarted whenever you make changes to your code.

First, clone the [`nodejs-mongo-mongoose` repository](https://github.com/do-community/nodejs-mongo-mongoose) from the [DigitalOcean Community GitHub account](https://github.com/do-community). This repository includes the code from the setup described in [How To Integrate MongoDB with Your Node Application](https://www.digitalocean.com/community/tutorials/how-to-integrate-mongodb-with-your-node-application). 

Clone the repository into a directory called `<^>node_project<^>`:

```command
git clone https://github.com/do-community/nodejs-mongo-mongoose.git <^>node_project<^>
```
Navigate to the `<^>node_project<^>` directory:

```command
cd <^>node_project<^>
```
Open the project's `package.json` file using `nano` or your favorite editor:

```command
nano package.json
```
Beneath the project dependencies and above the closing curly brace, create a new `devDependencies` object that includes `nodemon`:

```
[label ~/node_project/package.json]
...
"dependencies": {
    "ejs": "^2.6.1",
    "express": "^4.16.4",
    "mongoose": "^5.4.10"
  }<^>,<^>
  <^>"devDependencies": {<^>
    <^>"nodemon": "^1.18.10"<^>
  <^>}<^>    
}
```
Save and close the file when you are finished editing.

With the project code in place and its dependencies modified, you can move on to refactoring the code for a containerized workflow. 

## Step 2 — Modifying Your Code to Work with Containers

Modifying our application for a containerized workflow means making our code more modular. Containers offer portability between environments, and our code should reflect that by remaining as decoupled from the underlying operating system as possible. To achieve this, we will refactor our code to make greater use of Node's [process.env](https://nodejs.org/api/process.html#process_process_env) property, which returns an object with information about your user environment at runtime. We can use this object in our code to dynamically assign configuration information at runtime with environment variables. 

Let's begin with `app.js`, our main application entrypoint. Open the file:

```command
nano app.js
```
Inside, you will see a definition for a `port` [constant](https://www.digitalocean.com/community/tutorials/understanding-variables-scope-hoisting-in-javascript#constants), as well a [`listen` function](https://expressjs.com/en/4x/api.html#app.listen) that uses this constant to specify the port the application will listen on:

```
[label ~/home/node_project/app.js]
...
const port = 8080;
...
app.listen(port, function () {
  console.log('Example app listening on port 8080!');
});
```
Let's redefine the `port` constant to allow for dynamic assignment at runtime using the `process.env` object. Make the following changes to the constant definition and `listen` function:

```
[label ~/home/node_project/app.js]
...
<^>const port = process.env.PORT || 8080;<^>
...
app.listen(port, function () {
  console.log(<^>`Example app listening on ${port}!`<^>);
});
```
Our new constant definition assigns `port` dynamically using the value passed in at runtime or `8080`. Similarly, we've rewritten the `listen` function to use a [template literal](https://www.digitalocean.com/community/tutorials/how-to-work-with-strings-in-javascript#string-literals-and-string-values), which will interpolate the port value when listening for connections. Because we will be mapping our ports elsewhere, these revisions will prevent our having to continuously revise this file as our environment changes. 

Next, we will modify our database connection information to remove any configuration credentials. Open the `db.js` file, which contains this information:

```command
nano db.js
```
Currently, the file does the following things:
- Imports [Mongoose](https://mongoosejs.com/), the *Object Document Mapper* (ODM) that we're using to create schemas and models for our application data.
- Sets the database credentials as constants, including the username and password.
- Connects to the database using the [`mongoose.connect` method](https://mongoosejs.com/docs/api.html#connection_Connection).

For more information about the file, please see [Step 3](https://www.digitalocean.com/community/tutorials/how-to-integrate-mongodb-with-your-node-application#step-3-—-creating-mongoose-schemas-and-models) of [How To Integrate MongoDB with Your Node Application](https://www.digitalocean.com/community/tutorials/how-to-integrate-mongodb-with-your-node-application).

Our first step in modifying the file will be redefining the constants that include sensitive information. Currently, these constants look like this:

```
[label ~/node_project/db.js]
...
const MONGO_USERNAME = '<^>sammy<^>';
const MONGO_PASSWORD = '<^>your_password<^>';
const MONGO_HOSTNAME = '127.0.0.1';
const MONGO_PORT = '27017';
const MONGO_DB = '<^>sharkinfo<^>';
...
```
Instead of hardcoding this information, you can use the `process.env` object to capture the runtime values for these constants. Modify the block to look like this:

```
[label ~/node_project/db.js]
...
const {
  MONGO_USERNAME,
  MONGO_PASSWORD,
  MONGO_HOSTNAME,
  MONGO_PORT,
  MONGO_DB
} = process.env;
...
```
Our next step will be to make the connection method more robust by adding code that handles cases where our application fails to connect to our database. 

Currently, our [Mongoose `connect`](https://mongoosejs.com/docs/api.html#mongoose_Mongoose-connect) method accepts an option that tells Mongoose to use Mongo's [new URL parser](https://mongoosejs.com/docs/deprecations.html). Let's add a few more options to this method to define parameters for reconnection attempts. We can do this by creating an `options` constant that includes the relevant information, along with the new URL parser option. Below your Mongo constants, add the following definition for an `options` constant:

```
[label ~/node_project/db.js]
...
const {
  MONGO_USERNAME,
  MONGO_PASSWORD,
  MONGO_HOSTNAME,
  MONGO_PORT,
  MONGO_DB
} = process.env;

<^>const options = {<^>
  <^>useNewUrlParser: true,<^>
  <^>reconnectTries: Number.MAX_VALUE,<^>
  <^>reconnectInterval: 500,<^> 
  <^>connectTimeoutMS: 10000,<^>
<^>};<^>
...
```
The `reconnectTries` option tells Mongoose to continue trying to connect, while `reconnectInterval` defines the period between connection attempts in milliseconds. `connectTimeoutMS` defines 10 seconds as the period that the Mongo driver will wait before failing the connection attempt. 

We can now use the new `options` constant in the Mongoose `connect` method to fine tune our Mongoose connection settings. We will also add a [promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) to handle potential connection errors.

Currently, the Mongoose `connect` method looks like this:

```
[label ~/node_project/db.js]
...
const url = `mongodb://${MONGO_USERNAME}:${MONGO_PASSWORD}@${MONGO_HOSTNAME}:${MONGO_PORT}/${MONGO_DB}?authSource=admin`;

mongoose.connect(url, {useNewUrlParser: true});
```
Delete the existing `connect` method and replace it with the following code, which includes the `options` constant and a promise:

```
[label ~/node_project/db.js]
...
const url = `mongodb://${MONGO_USERNAME}:${MONGO_PASSWORD}@${MONGO_HOSTNAME}:${MONGO_PORT}/${MONGO_DB}?authSource=admin`;

<^>mongoose.connect(url, options).then( function() {<^>
  <^>console.log("MongoDB is connected");<^>
<^>})<^>
  <^>.catch( function(err) {<^>
  <^>console.log(err);<^>
<^>});<^>
```
In the case of a successful connection, our function logs an appropriate message; otherwise it will [`catch`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch) and log the error, allowing us to troubleshoot.

Save and close the file when you have finished editing.

You have now rewritten your code to remove sensitive information and fine tune your database connection settings and error handling. The next step will be to create environment variables to pass the required information to the application at runtime.

## Step 3 — Adding Environment Variables

At this point, we have modified `db.js` to work with environment variables, but we still need a way to pass these variables to our application. To do this, we will create two files: one for our Node environment variables, and another that will create a user on the database instance using the [environment variables](https://docs.docker.com/samples/library/mongo/#environment-variables) that the [Mongo Docker image](https://hub.docker.com/_/mongo) makes available at runtime.

Let's start by the creating an `.env` file for our Node variables.

Open the file:

```command
nano .env
```
This file will include the information that we removed from `db.js`: the username and password for our application's database, and the host and port settings. Remember to update the username, password, and database name listed here with your own information:

```
[label ~/node_project/.env]
MONGO_USERNAME=<^>sammy<^>
MONGO_PASSWORD=<^>your_password<^>
MONGO_HOSTNAME=db
MONGO_PORT=27017
MONGO_DB=<^>sharkinfo<^>
```
Note that the host here is `db`, and not `127.0.0.1`, as it was in our original `db.js` file. Our host in this case will be the Mongo `db` instance, which we will create when we write our `docker-compose.yml` file.

Save and close this file when you are finished editing.

Our next step will be to make a file called `db-init.js`, which will create an authenticated user on our Mongo instance. The user we enter here will be the same user we defined for our Node application in the `.env` file.

This file will make implicit use of the environment variables that the Mongo image makes available at runtime. These include: 
- `MONGO_INITDB_ROOT_USERNAME`: This sets a new user in the [`admin` authentication database](https://docs.mongodb.com/manual/core/security-users/#user-authentication-database).
- `MONGO_INITDB_ROOT_PASSWORD`: This sets the password for that user.
- `MONGO_INITDB_DATABASE`: This specifies the database that `mongo` will use when executing any scripts defined in the `docker-entrypoint-initdb.d` directory. If there are scripts defined in that directory and the database is not defined, `mongo` will default to using the `test` database.

You will see these variables again when we define the `db` service in the `docker-compose.yml` file. 

Our `db-init.js` file will assume that we have created a root user and password in `docker-compose.yml`, ensuring that our MongoDB instance will start with authentication enabled. It will also assume that we have specified the `admin` database as the database we want to use to execute the file. 

Open `db-init.js`:

```command
nano db-init.js
```
In the file, add the following code to create a user with a specific password and [`readWriteAny`](https://docs.mongodb.com/manual/reference/built-in-roles/#readWriteAnyDatabase) privileges on all databases except `local` and `config`. Remember to replace the information listed here with your own information, and make sure that the user and password you define in this file match the information you entered in your `.env` file:

```
[label ~/node_project/db-init.js]
db.createUser({
  user: '<^>sammy<^>',
  pwd: '<^>your_password<^>',
  roles: [
    {
      role: 'readWriteAnyDatabase',
      db: 'admin'
    },
  ],
});
```
Our `<^>sammy<^>` user will now have `readWriteAny` privileges on the `<^>sharkinfo<^>` database that we defined in our `.env` file, [authenticated](https://docs.mongodb.com/manual/reference/connection-string/#urioption.authSource) by the `admin` database. 

Save and close the file when you are finished editing. 

Because your `.env` and `db-init.js` files contain sensitive information, you will want to add them to the project's `.dockerignore` and `.gitignore` files so that they do not copy to your version control or containers. 

Open your `.dockerignore` file:

```command
nano .dockerignore
```
Add the following lines to the bottom of the file:

```
[label ~/node_project/.dockerignore]
...
.gitignore
<^>.env<^>
<^>db-init.js<^>
```
Save and close the file when you are finished editing.

Open `.gitignore`:

```command
nano .gitignore
```
In this case, the file already specifies `.env`, so we'll just add `db-init.js` to the bottom of the file:

```
[label ~/node_project/.gitignore]
...
.dynamodb/ 
<^>db-init.js<^>
```
Save and close the file when you are finished editing.

As a final step, we will move our application code to a separate directory to ensure that our sensitive files do not get mounted to our container. Because our setup will use [bind mounts](https://docs.docker.com/storage/bind-mounts/) to keep the application code on the host synchronized with code in the container, we run the risk of overriding the security precautions we've taken in our `.dockerignore` file when the application directory is mounted to the container. Moving our application files and leaving our Docker-related and configuration files at the root of the project will prevent any unintended copying.

Make a new directory called `app` for the application code:

```command
mkdir app
```
Move `app.js`, `db.js`, `package.json`, `package-lock.json` and the `controllers`, `models`, `routes`, and `views` directories into the `app` directory:

```command
mv app.js db.js *.json controllers models routes views -t app
```
You have now successfully separated sensitive information from your project code and taken measures to control how and where this information gets copied. It's time to create your service definitions with Compose.

## Step 4 — Defining Services with Docker Compose

With your code refactored, you are ready to write the `docker-compose.yml` file that will contain the service definitions you will use to run your containers. Before defining these services, however, let's make sure our application's `Dockerfile` reflects the new location of our application code. We will also create a script using the [`wait-for`](https://github.com/Eficode/wait-for) tool to ensure that our containers start in the right order.

Begin by opening the application `Dockerfile`:

```command
nano Dockerfile
```
This file contains instructions for building the application image, including copying the application code, installing the project dependencies, and running the container as a non-root user. Rather than copying the code from the root project directory, `~/<^>node_project<^>`, we want the instructions to copy from `~/<^>node_project<^>/app`. 

Edit the following `COPY` instructions in the file to specify the new location of your application code:

```
[label ~/node_project/Dockerfile]
...
COPY <^>app/<^>package*.json ./
...
COPY --chown=node:node .<^>/app<^> .
```
The revised `Dockerfile` will look like this:

```
[label ~/node_project/Dockerfile]
FROM node:10-alpine

RUN mkdir -p /home/node/app/node_modules && chown -R node:node /home/node/app

WORKDIR /home/node/app

COPY app/package*.json ./

USER node

RUN npm install

COPY --chown=node:node ./app .

EXPOSE 8080

CMD [ "node", "app.js" ]
```
Save and close the file when you are finished editing.

Next, you will create a script that will allow you to control the order in which your containers are started. The `wait-for` tool is a wrapper script that allows you to poll whether or not a specific host and port are accepting TCP connections. Though Compose allows you to specify dependencies between services using the [`depends_on` option](https://docs.docker.com/compose/compose-file/#depends_on), this order is based on whether or not the container is running rather than its readiness. This could create complications for our setup: even if we specify that our Node application depends on our database service, the application service will not necessarily wait until all of the database startup tasks are complete to attempt to connect.

You have already built some resiliency into your application code by adding retry options and a promise and error handler to the connection method in `db.js`. Using the `wait-for` tool will allow you an even greater degree of control over attempts to connect to the database by testing whether or not the database is in fact ready. For more information on `wait-for` and other tools that can control startup order, please see the relevant [recommendations in the Compose documentation](https://docs.docker.com/compose/startup-order/).

In the `app` folder, open a file called `wait-for.sh`:

```command
nano app/wait-for.sh
```
Paste the following code into the file to create the polling function:

```
[label ~/node_project/app/wait-for.sh]
#!/bin/sh

TIMEOUT=15
QUIET=0

echoerr() {
  if [ "$QUIET" -ne 1 ]; then printf "%s\n" "$*" 1>&2; fi
}

usage() {
  exitcode="$1"
  cat << USAGE >&2
Usage:
  $cmdname host:port [-t timeout] [-- command args]
  -q | --quiet                        Do not output any status messages
  -t TIMEOUT | --timeout=timeout      Timeout in seconds, zero for no timeout
  -- COMMAND ARGS                     Execute command with args after the test finishes
USAGE
  exit "$exitcode"
}

wait_for() {
  for i in `seq $TIMEOUT` ; do
    nc -z "$HOST" "$PORT" > /dev/null 2>&1
    
    result=$?
    if [ $result -eq 0 ] ; then
      if [ $# -gt 0 ] ; then
        exec "$@"
      fi
      exit 0
    fi
    sleep 1
  done
  echo "Operation timed out" >&2
  exit 1
}

while [ $# -gt 0 ]
do
  case "$1" in
    *:* )
    HOST=$(printf "%s\n" "$1"| cut -d : -f 1)
    PORT=$(printf "%s\n" "$1"| cut -d : -f 2)
    shift 1
    ;;
    -q | --quiet)
    QUIET=1
    shift 1
    ;;
    -t)
    TIMEOUT="$2"
    if [ "$TIMEOUT" = "" ]; then break; fi
    shift 2
    ;;
    --timeout=*)
    TIMEOUT="${1#*=}"
    shift 1
    ;;
    --)
    shift
    break
    ;;
    --help)
    usage 0
    ;;
    *)
    echoerr "Unknown argument: $1"
    usage 1
    ;;
  esac
done

if [ "$HOST" = "" -o "$PORT" = "" ]; then
  echoerr "Error: you need to provide a host and port to test."
  usage 2
fi

wait_for "$@"
```
Save and close the file when you are finished adding the code.

Make the script executable:

```command
chmod +x app/wait-for.sh
```

Next, open the `docker-compose.yml` file:

```command
nano docker-compose.yml
```
First, define the `nodejs` application service by adding the following code to the file:

```
[label ~/node_project/docker-compose.yml]
version: '3'

services:
  nodejs:
    build:
      context: .
      dockerfile: Dockerfile
    image: nodejs
    container_name: nodejs
    restart: unless-stopped
    env_file: .env
    environment:
      - MONGO_USERNAME=$MONGO_USERNAME
      - MONGO_PASSWORD=$MONGO_PASSWORD
      - MONGO_HOSTNAME=$MONGO_HOSTNAME
      - MONGO_PORT=$MONGO_PORT
      - MONGO_DB=$MONGO_DB 
    ports:
      - "80:8080"
    volumes:
      - ./app:/home/node/app
      - node_modules:/home/node/app/node_modules
    depends_on:
      - db
    networks:
      - app-network
    command: ./wait-for.sh db:27017 -- /home/node/app/node_modules/.bin/nodemon app.js
```
The `nodejs` service definition includes the following options:
- `build`: This defines the configuration options, including the `context` and `dockerfile`, that will be applied when Compose builds the application image. If you wanted to use an existing image from a registry like [Docker Hub](https://hub.docker.com/), you could use the [`image` instruction](https://docs.docker.com/compose/compose-file/#image) instead, with information about your username, repository, and image tag.
- `context`: This defines the build context for the image build — in this case, the current project directory.
- `dockerfile`: This specifies the `Dockerfile` that you modified at the beginning of this Step as the file Compose will use to build the application image.
- `image`, `container_name`: These apply names to the image and container.
- `restart`: This defines the restart policy. The default is `no`, but we have set the container to restart unless it is stopped.
- `env_file`: This tells Compose that we would like to add environment variables from a file called `.env`, located in our build context.
- `environment`: Using this option allows you to add the Mongo connection settings you defined in the `.env` file. Note that we are not setting `NODE_ENV` to `development`, since this is [Express's](https://expressjs.com/) [default](https://github.com/expressjs/express/blob/dc538f6e810bd462c98ee7e6aae24c64d4b1da93/lib/application.js#L71) behavior if `NODE_ENV` is not set. When moving to production, you can set this to `production` to [enable view caching and less verbose error messages](https://expressjs.com/en/advanced/best-practice-performance.html#set-node_env-to-production). 
- `ports`: This maps port `80` on the host to port `8080` on the container.
- `volumes`: We are including two types of mounts here:
    - The first is a [bind mount](https://docs.docker.com/storage/bind-mounts/) that mounts our application code on the host to the `/home/node/app` directory on the container. This will facilitate rapid development, since any changes you make to your host code will be populated immediately in the container.
    - The second is a named [volume](https://docs.docker.com/storage/volumes/), `node_modules`. We are using this to circumvent the behavior of the bind mount, which will hide the newly created `node_modules` directory on the container. Since `node_modules` on the host is empty, this will map an empty directory to the container, preventing our application from starting. Instead, we are creating a named volume that persists the contents of the `/home/node/app/node_modules` directory and mounts it to the container,
    hiding the bind.

    **Keep the following points in mind when using this approach**: 
    - Your bind will mount the contents of the `node_modules` directory on the container to the host and this directory will be owned by `root`, since the named volume was created by Docker. 
    - If you have a pre-existing `node_modules` directory on the host, it will override the `node_modules` directory created on the container. The setup that we're building in this tutorial assumes that you do **not** have a pre-existing `node_modules` directory and that you won't be working with `npm` on your host. This is in keeping with a [twelve-factor approach to application development](https://12factor.net/), which minimizes dependencies between execution environments.

- `depends_on`: This options specifies the order in which we would like Compose to start our containers. Our `wait-for` script will address any potential discrepancies by preventing the application service from attempting to connect before the database service is ready. 
- `networks`: This specifies that our application service will join the `app-network` network, which we will define at the bottom on the file.
- `command`: This option lets you set the command that should be executed when Compose runs the image. Note that this will override the `CMD` instruction that we set in our application `Dockerfile`. Here, we are running the application using the `wait-for` script, which will poll the `db` service on port `27017` to test whether or not the database service is ready. Once the readiness test is successful, the script will execute the command we have set, `/home/node/app/node_modules/.bin/nodemon app.js`, to start the application with `nodemon`. This will ensure that any future changes we make to our code are reloaded without our having to restart the application.

Next, create the `db` service by adding the following code below the application service definition:

```
[label ~/node_project/docker-compose.yml]
...
  db:
    image: mongo
    container_name: db
    restart: unless-stopped
    environment:
      - MONGO_INITDB_ROOT_USERNAME=<^>user<^>
      - MONGO_INITDB_ROOT_PASSWORD=<^>password<^>
      - MONGO_INITDB_DATABASE=admin
    volumes:  
      - ./db-init.js:/docker-entrypoint-initdb.d/db-init.js
      - dbdata:/data/db   
    networks:
      - app-network  
```
Some of the settings we defined for the `nodejs` service remain the same, but we've also made the following changes to the `image`, `environment`, and `volumes` definitions: 
- `image`: To create this service, Compose will pull the [Mongo image](https://hub.docker.com/_/mongo) from Docker Hub. 
- `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`, `MONGO_INITDB_DATABASE`: The `USERNAME` and `PASSWORD` variables will create a `root` user, here called `<^>user<^>`, in the `admin` database, ensuring that authentication is enabled when the container starts. Including the `DATABASE` variable tells Mongo the name of the database you want to use for the user creation script you saved to `db-init.js` in [Step 3](). Remember to replace the username and password values listed here with your own information.
- `./db-init.js:/docker-entrypoint-initdb.d/db-init.js`: This bind mounts the `db-init.js` script to the `docker-entrypoint-initdb.d` directory on the container, where Mongo will look for creation scripts.
- `dbdata:/data/db`: The named volume `dbdata` will persist the data stored in Mongo's [default data directory](https://docs.mongodb.com/manual/reference/configuration-options/#storage.dbPath), `/data/db`. This will ensure that you don't lose data in cases where you stop or remove containers.

We've also added the `db` service to the `app-network` network with the `networks` option.

As a final step, add the volume and network definitions to the bottom of the file:

```
[label ~/node_project/docker-compose.yml]
...
networks:
  app-network:
    driver: bridge

volumes:
  dbdata:
  node_modules:  
```
The user-defined bridge network `app-network` enables communication between our containers since they are on the same Docker daemon host. This streamlines traffic and communication within the application, as it opens all ports between containers on the same bridge network, while exposing no ports to the outside world. Thus, our `db` and `nodejs` containers can communicate with each other, and we only need to expose port `80` for front-end access to the application.

Finally, we've added our named volume definitions for the `dbdata` and `node_modules` volumes.

The finished `docker-compose.yml` file will look like this:

```
[label ~/node_project/docker-compose.yml]
version: '3'

services:
  nodejs:
    build:
      context: .
      dockerfile: Dockerfile
    image: nodejs
    container_name: nodejs
    restart: unless-stopped
    env_file: .env
    environment:
      - MONGO_USERNAME=$MONGO_USERNAME
      - MONGO_PASSWORD=$MONGO_PASSWORD
      - MONGO_HOSTNAME=$MONGO_HOSTNAME
      - MONGO_PORT=$MONGO_PORT
      - MONGO_DB=$MONGO_DB
    ports:
      - "80:8080"
    volumes:
      - ./app:/home/node/app
      - node_modules:/home/node/app/node_modules
    depends_on:
      - db
    networks:
      - app-network
    command: ./wait-for.sh db:27017 -- /home/node/app/node_modules/.bin/nodemon app.js 

  db:
    image: mongo
    container_name: db
    restart: unless-stopped
    environment:
      - MONGO_INITDB_ROOT_USERNAME=<^>user<^>
      - MONGO_INITDB_ROOT_PASSWORD=<^>password<^>
      - MONGO_INITDB_DATABASE=admin
    volumes:     
      - ./db-init.js:/docker-entrypoint-initdb.d/db-init.js
      - dbdata:/data/db
    networks:
      - app-network  

networks:
  app-network:
    driver: bridge

volumes:
  dbdata:
  node_modules:  
```
Save and close the file when you are finished editing.

With your service definitions in place, you are ready to start the application.

Create the services with [`docker-compose up`](https://docs.docker.com/compose/reference/up/) and the `-d` flag, which will run the `nodejs` and `db` containers in the background:

```command
docker-compose up -d
```
You will see output confirming that your services have been created: 

```
[secondary_label Output]
...
Creating db ... done
Creating nodejs ... done
```
You can also get more detailed information about the startup processes by displaying the log output from the services:

```command
docker-compose logs 
```
You will see something like this if everything has started correctly:

```
[secondary_label Output]
...
nodejs    | [nodemon] starting `node app.js`
nodejs    | Example app listening on 8080!
nodejs    | MongoDB is connected
...
db        | 2019-02-22T17:26:27.329+0000 I ACCESS   [conn2] Successfully authenticated as principal <^>sammy<^> on admin
```
You can also check the status of your containers with [`docker-compose ps`](https://docs.docker.com/compose/reference/ps/):

```command
docker-compose ps
```
You will see output indicating that your containers are running:

```
[secondary_label Output]
 Name               Command               State          Ports        
----------------------------------------------------------------------
db       docker-entrypoint.sh mongod      Up      27017/tcp           
nodejs   ./wait-for.sh db:27017 --  ...   Up      0.0.0.0:80->8080/tcp
```

With your services running, you can visit `http://<^>your_server_ip<^>` in the browser. You will see a landing page that looks like this:

![Application Landing Page](https://assets.digitalocean.com/articles/docker_node_image/landing_page.png)

Click on the **Get Shark Info** button. You will see a page with an entry form where you can enter a shark name and a description of that shark's general character:

![Shark Info Form](https://assets.digitalocean.com/articles/node_mongo/shark_form.png)

In the form, add a shark of your choosing. For the purpose of this demonstration, we will add `<^>Megalodon Shark<^>` to the **Shark Name** field, and `<^>Ancient<^>` to the **Shark Character** field:

![Filled Shark Form](https://assets.digitalocean.com/articles/node_mongo/shark_filled.png)

Click on the **Submit** button. You will see a page with this shark information displayed back to you:

![Shark Output](https://assets.digitalocean.com/articles/node_mongo/shark_added.png)

As a final step, we can test that the data you've just entered will persist if you remove your database container.

Back at your terminal, type the following command to stop and remove your containers and network:

```command
docker-compose down
```
Note that we are *not* including the `--volumes` option; hence, our `dbdata` volume is not removed. 

The following output confirms that your containers and network have been removed:

```
[secondary_label Output]
Stopping nodejs ... done
Stopping db     ... done
Removing nodejs ... done
Removing db     ... done
Removing network node_project_app-network
```
Recreate the containers:

```command
docker-compose up -d
```
Now head back to the shark information form:

![Shark Info Form](https://assets.digitalocean.com/articles/node_mongo/shark_form.png)

Enter a new shark of your choosing. We'll go with `<^>Whale Shark<^>` and `<^>Large<^>`:

![Enter New Shark](https://assets.digitalocean.com/articles/node_docker_dev/whale_shark.png)

Once you click **Submit**, you will see that the new shark has been added to the shark collection in your database without the loss of the data you've already entered:

![Complete Shark Collection](https://assets.digitalocean.com/articles/node_docker_dev/persisted_data.png)

Your application is now running on Docker containers with data persistence and code synchronization enabled. 

## Conclusion

You now have a development setup for your Node application running on Docker containers. You've made your project more modular and portable by extracting sensitive information and decoupling your application's state from your application code. You have also configured a boilerplate `docker-compose.yml` file that you can revise as your development needs change. 

As you develop, you may be interested in learning more about designing applications for containerized and [Cloud Native](https://github.com/cncf/toc/blob/master/DEFINITION.md) workflows. Please see [Architecting Applications for Kubernetes](https://www.digitalocean.com/community/tutorials/architecting-applications-for-kubernetes) and [Modernizing Applications for Kubernetes](https://www.digitalocean.com/community/tutorials/modernizing-applications-for-kubernetes) for more information on these topics.

To learn more about the code used in this tutorial, please see [How To Build a Node.js Application with Docker](https://www.digitalocean.com/community/tutorials/how-to-build-a-node-js-application-with-docker) and [How To Integrate MongoDB with Your Node Application](https://www.digitalocean.com/community/tutorials/how-to-integrate-mongodb-with-your-node-application). For information about deploying a Node application with an [Nginx](https://www.nginx.com/) reverse proxy using containers, please see [How To Secure a Containerized Node.js Application with Nginx, Let's Encrypt, and Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose).


