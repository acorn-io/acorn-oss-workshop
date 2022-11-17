Now that we have seen that the application works well with Acorn, we will use another feature of Acorn which allows us to package the app as a single artifact and push it to the Docker Hub.

As a reminder, below is the Acornfile we defined up till now:

```
containers: {
  
  voteui: {
    build: "./vote-ui"
    ports: publish : "80/http"
  }

  vote: {
    build: "./vote"
    ports: "5000/http"
  }

  redis: {
    image: "redis:7.0.5-alpine3.16"
    ports: "6379/tcp"
    dirs: {
      "/data": "volume://redis"
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
      "/var/lib/postgresql/data*": "volume://db"
    }
  }

  result: {
    build: "./result"
    ports: "5000/http"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
   }
  }

  resultui: {
    build: "./result-ui"
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
    "db": {
        size: 100M
    }
    "redis": {
        size: 100M
    }
}
```

## Publishing to Docker Hun

First we log into the Docker Hub (use your own DockerHub credentials or create them if you don't have a DockerHub account yet):

Note: in the commands below replace YOUR_DOCKERHUB_USERNAME with your real username

```
$ acorn login docker.io
? Username YOUR_DOCKERHUB_USERNAME
? Password **************
index.docker.io
```

Next we build the application, this will result into the creation of a single OCI image:

```
acorn build -t docker.io/YOUR_DOCKERHUB_USERNAME/vote-workshop:v1.0.0 .
```

Then we push it to the registry (docker.io relates to the Docker Hub)

```
acorn push docker.io/YOUR_DOCKERHUB_USERNAME/vote-workshop:v1.0.0 .
```

From the Docker Hub we verify the new image is there:

TODO: screenshot

Your version of the VotingApp is now available in the DockerHub and it can be used by anyone using the following command:

```
acorn run --name webhooks docker.io/YOUR_DOCKERHUB_USERNAME/vote-workshop:v1.0.0 .
```
