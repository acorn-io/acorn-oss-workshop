In the previous step we enhanced the Acornfile of the VotingApp defining volumes for both *db* and *redis* containers. In this step we will use Acorn's development mode which eases the development workflow.

## About development mode

In development mode, Acorn allows to make changes to the source code and see it updated inside the app containers in real time. In this mode Acorn will watch the local directory for changes and synchronize them to the running Acorn app. To activate the development mode we need to use the *-i* flag when running the app as you will do in a bit.

In this step we will focus on the *result* microservice which is developed with *Node.Js*. If you have a look into the folder containing the application code, you will see 2 build targets are defined in the Dockerfile: the first one is named *dev*, the second one *production*:

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

- when the image is built for the *dev* target, the command ```npm run dev``` runs *nodemon* under the hood. nodemon is a process which watches the code changes and which is able to relaunch the application
- when the image is built for the *production* target, the command ```npm start``` runs the standard node binary. No hot reload is possible in that case

Before running the Acorn application in development mode we need to modify the Acornfile is bit to make sure if the dev mode is detected:
- the code folder of the result microservice is mounted into the */app* folder within the container
- the build is done against the *dev* target, this will ensure nodemon is the main process running in the result container thus making hot reload possible

Modify the definition of the *result* container so it looks as follows:

```
result: {
  build: {
    target: std.ifelse(args.dev, "dev", "production")
    context: "./result"
  }
  if args.dev {
    dirs: {
        "/app": "./result"
    }
  }   
  ports: "5000/http"
  env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
}
```

Note: Acorn provides many useful functions such as the *std.ifelse*, an helper to perform an if...else...end statement on a single line. All the function available are listed in the [function library documentation](https://docs.acorn.io/reference/functions)

You can now update the application running it in development mode:

```
acorn run -n vote -i .
```

note: in development mode you'll notice that the logs of each containers are streamed to the console

Once the application is up and running you can modify the *server.js* file located in the *result* folder and see that changes automatically been taken into account making the container's process to restart.

Below is an example of logs you can get for the result service following a change in server.js:

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

Change the Acornfile modifying the definition of the *voteui*, *vote* and *result-ui* containers to ensure the development mode is working fine for those ones as well:

- voteui:

```
voteui: {
  if args.dev {
    dirs: {
      "/usr/share/nginx/html": "./vote-ui"
    }
  }
  build: {
    context: "./vote-ui"
  }
  ports: publish : "80/http"
}
```

- vote:

```
vote: {
  build: {
    target: std.ifelse(args.dev, "dev", "production")
    context: "./vote"
  }
  if args.dev {
    dirs: {
        "/app": "./vote"
    }
  }
  ports: "5000/http"
}
```

- resultui:

```
resultui: {
  build: {
    target: std.ifelse(args.dev, "dev", "production")
    context: "./result-ui"
  }
  if args.dev {
    dirs: {
      "/app": "./result-ui"
    }
  } 
  ports: publish : "80/http"
}
```

Once you have modified the Acornfile you can update the application one more time:

```
acorn run -n vote -i . 
```

We can go one step further and make sure volumes are not used when the app is run in development mode.  
In order to do that we need to:

- add a condition in the *volumes* key so that no volumes are created in dev mode:

```
volumes: {
  if !args.dev {
    "db": {
        size: "100M"
    }
    "redis": {
        size: "100M"
    }
  }
}
```

- add a condition in *redis* container so content of */data* is not persisted in a volume while in dev mode:

```
redis: {
  labels: {
    component: "redis"
  }
  image: "redis:7.0.5-alpine3.16"
  ports: "6379/tcp"
  dirs: {
    if !args.dev {
      "/data": "volume://redis"
    }
  }
}
```

- add a condition in *db* container so content of */var/lib/postgresql/data* is not persisted in a volume while in dev mode:

```
db: {
  labels: {
    component: "db"
  }
  image: "postgres:15.0-alpine3.16"
  ports: "5432/tcp"
  env: {
    "POSTGRES_USER": "secret://db-creds/username"
    "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
  dirs: {
    if !args.dev {
      "/var/lib/postgresql/data": "volume://db"
    }
  }
}
```

Make sure the VotingApp is working fine in development mode using this new version of the Acornfile. 

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
```
containers: {
  voteui: {
    if args.dev {
      dirs: {
        "/usr/share/nginx/html": "./vote-ui"
      }
    }
    build: {
      context: "./vote-ui"
    }
    ports: publish : "80/http"
  }

  vote: {
    build: {
      target: std.ifelse(args.dev, "dev", "production")
      context: "./vote"
    }
    if args.dev {
      dirs: {
          "/app": "./vote"
      }
    }
    ports: "5000/http"
  }
  
  redis: {
    image: "redis:7.0.5-alpine3.16"
    ports: "6379/tcp"
    dirs: {
      if !args.dev {
        "/data": "volume://redis"
      }
    }
  }

  worker: {
    build: "./worker/go"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
  }

  db: {
    image: "postgres:15.0-alpine3.16"
    ports: "5432/tcp"
    env: {
      "POSTGRES_USER": "secret://db-creds/username"
      "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
    dirs: {
      if !args.dev {
        "/var/lib/postgresql/data": "volume://db"
      }
    }
  }

  result: {
    build: {
      target: std.ifelse(args.dev, "dev", "production")
      context: "./result"
    }
    if args.dev {
      dirs: {
          "/app": "./result"
      }
    }   
    ports: "5000/http"
    env: {
      "POSTGRES_USER": "secret://db-creds/username"
      "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
  }

  resultui: {
    build: {
      target: std.ifelse(args.dev, "dev", "production")
      context: "./result-ui"
    }
    if args.dev {
      dirs: {
        "/app": "./result-ui"
      }
    } 
    ports: publish : "80/http"
  }
}

secrets: {
    "db-creds": {
        type: "basic"
        data: {
            username: ""
            password: ""
        }
    }
}

volumes: {
  if !args.dev {
    "db": {
        size: "100M"
    }
    "redis": {
        size: "100M"
    }
  }
}
```
</details>

Note: you can find more information about development mode in [the official documentation](https://docs.acorn.io/getting-started#step-6-development-mode)

[Previous](./volumes.md)  
[Next](./profiles.md)