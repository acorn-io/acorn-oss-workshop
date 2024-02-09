Defining memory contraints for each container is a the best practice as it allows to have more control on over application. This could prevent resources exhaustion in the case a container start acting strangely allocating too much memory.

Acorn offers a way to specify memory constraints using the *memory* property in the definition of a container. For instance, we can specify the worker container cannot use more than 32Mi of memory:

```
  worker: {
    build: "./worker/go"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
    memory: 32Mi
  }
```

Note: the amount of memory used here is arbitrary, in a real world scenario we would first observe how the application behaves with time (for instance using a monitoring stack based on Prometheus) and set the memory contraint accordingly.

Memory constraints can also be provided when running an acorn:

```
acorn run -n vote -m worker=32Mi .
```

Setup the same memory contraints of *32Mi* to all the containers of the VotingApp (*voteui*, *vote*, *worker*, *result*, *resultui*, *redis* and *pg*) and run the application:

```
acorn run -n vote --update .
```

Listing the containers you should see a couple of them (*result* and *vote* and probably *db* are not running correctly)

```
acorn containers
```

Under the hood we could see those containers get an *OOMKilled* error because they reach the memory limit.

To make sure each container have enough resource, change the value of the *memory* property to *128Mi* for all the containers. Once you have tested that the application is running fine you can remove the app and all the related resources:

```
acorn rm vote --all --force
```

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
<pre>
containers: {
  voteui: {
    build: "./vote-ui"
    ports: publish : "80/http"
    memory: 128Mi
  }
  vote: {
    build: "./vote"
    ports: "5000/http"
    memory: 128Mi
  }
  redis: {
    image: "redis:7.0.5-alpine3.16"
    ports: "6379/tcp"
    dirs: {
      "/data": "volume://redis"
    }
    memory: 128Mi
  }
  worker: {
    build: "./worker/go"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
    memory: 128Mi
  }
  db: {
    image: "postgres:15.0-alpine3.16"
    ports: "5432/tcp"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
     "PGDATA": "/var/lib/postgresql/data/db"
    }
    dirs: {
      "/var/lib/postgresql/data": "volume://db"
    }
    memory: 128Mi
  }
  result: {
    build: "./result"
    ports: "5000/http"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
    memory: 128Mi
  }
  resultui: {
    build: "./result-ui"
    ports: publish : "80/http"
    memory: 128Mi
  }
}
secrets: {
    "db-creds": {
        type: "basic"
        params: {
          usernameLength:     7
          usernameCharacters: "a-z"
          passwordLength:     10
          passwordCharacters: "A-Za-z0-9"
        }
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
</pre>
</details>

Note: you can find more information about memory constraints in [the Acorn documentation](https://docs.acorn.io/reference/memory)

[Previous](./volumes.md)  
[Next](./development_mode.md)