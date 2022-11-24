In the previous step we enhanced the Acornfile of the VotingApp defining volumes for both db and redis containers. In this step we will use Acorn's development mode which eases the development workflow.

## About development mode

In development mode, Acorn allows to make changes to the source code and see it updated inside the app containers in real time. In this mode Acorn will watch the local directory for changes and synchronize them to the running Acorn app. To activate the development mode we need to use the *-i* flag when running the app as you will do in a bit.

In this step we will focus on the result microservice which is developed with *Node.js*. If you have a look into the folder containing the application code, you will see the following Dockerfile where 2 targets are defined: the second one is named *dev*, the second one *production*:

```
FROM node:18.12.1-slim as base
WORKDIR /app
COPY . .
EXPOSE 5000

FROM base as production
ENV NODE_ENV=production
RUN npm ci --production
CMD ["npm", "start"]

FROM base as dev
ENV NODE_ENV=development
RUN npm ci
CMD ["npm", "run", "dev"]
```

- when the image is built for the *dev* target, the command "npm run dev" runs *nodemon* under the hood. nodemon is a process which watches the code changes and is able to relaunch the application when changes occur in the container filesystem
- when the image is built for the *production* target, the command "npm start" runs the standard node binary to run the application. No hot reload is possible in that case

Before running the Acorn application in development mode we need to modify the Acornfile is bit to make sure that if the dev mode is detected:
- the code folder of the result microservice is mounted into the */app* folder within the container
- the build is done against the *dev* target, this will ensure nodemon is the main process running in the result container thus making hot reload possible

Modify the definition of the *result* container so it looks as follows:

```
result: {
  if args.dev {
    build: {
      target: "dev"
      context: "./result"
    }
    dirs: {
        "/app": "./result"
    }
  } 
  if !args.dev {
    build: {
      target: "production"
      context: "./result"
    }
  }  
  ports: "5000/http"
  env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
}
```

You can now stop the application (if it was already running) and run it in development mode:

```
acorn run -n vote -i .
```

note: in development mode you'll notice that the logs of each containers are streamed in the console.

Once the application is up and running you can modify the *server.js* file located in the *result* folder and see that changes automatically been taken into account making the container's process to restart.

Below is an example of logs we can get for the result service following a change in server.js:

```
result-f4fd75fb5-66mjc: (sync): Upstream - Handling 2 removes
result-f4fd75fb5-66mjc: (sync): Upstream - Remove '.server.js.swp'
result-f4fd75fb5-66mjc: (sync): Upstream - Remove '.server.js.swp'
result-f4fd75fb5-66mjc: (sync): Upstream - Remove '.server.js.swp'
result-f4fd75fb5-66mjc: (sync): Upstream - Upload File 'server.js'
result-f4fd75fb5-66mjc: (sync): Upstream - Upload 1 create change(s) (Uncompressed ~2.66 KB)
result-f4fd75fb5-66mjc: (sync): Upstream - Successfully processed 3 change(s)
result-f4fd75fb5-66mjc: [nodemon] restarting due to changes...
result-f4fd75fb5-66mjc: [nodemon] starting `node server.js`
result-f4fd75fb5-66mjc: This app is running on port 5000
result-f4fd75fb5-66mjc: Connected to db
result-f4fd75fb5-66mjc: new socket.io connection
...
```

We only show the development mode for the *result* microservice but the same principles would apply for the other microservices as well. 

As an additional exercise, change the Acornfile modifying the definition of the *vote* container to ensure the development mode is working fine for that one as well (the *vote* microservice is a Python flask application).

<details>
  <summary markdown="span">Solution</summary>

To work with the development mode, the definition of the *vote* container can be modified as follows:

```
vote: {
  if args.dev {
      build: {
        target: "dev"
        context: "./vote"
      }
      dirs: {
          "/app": "./vote"
      }
    } 
    if !args.dev {
      build: {
        target: "production"
        context: "./vote"
      }
    }  
    ports: "5000/http"
  }
}
```

</details>

We could also do the same with *vote-ui* and *result-ui* containers.

[Previous](./acorn_image.md)