Defining memory contraints for each container is a the best practice as it allows to have more control on over application. This could prevent resources exhaustion in the case a container start acting strangely allocating too much memory.

Acorn offers a way to specify memory constraints using the *memory* property in the definition of a container. For instance, we can specify the worker container cannot use more than 32Mi of memory:

```
  worker: {
    labels: {
      component: "worker"
    }
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

Change the value of the *memory* property to *128Mi* for *result*, *vote* and *db* containers and update the application one more time You should now see all containers are running fine.

In development mode *resultui* container needs much more memory than in normal mode. Make sure to add the memory constraint for that particular container only when the application is running in development mode. This can be done with the following statement:

```
  if ! args.dev {
      memory: 32Mi
  }
```

Once you have tested that the application is running fine you can remove the app and all the related resources:

```
acorn rm vote --all --force
```

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
<pre>
labels: {
    application: "votingapp"
}
args: {
    replicas: 3
}
profiles: {
    dev: {
        replicas: 1
    }
    test: {
        replicas: 2
    }
}
containers: {
  voteui: {
    labels: {
      component: "voteui"
    }
    if args.dev {
      dirs: "/usr/share/nginx/html": "./vote-ui"
    }
    build: {
      context: "./vote-ui"
    }
    ports: publish : "80/http"
    scale: args.replicas
    memory: 32Mi
  }
  vote: {
    labels: {
      component: "vote"
    }
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
    memory: 128Mi
  }
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
    memory: 32Mi
  }
  worker: {
    labels: {
      component: "worker"
    }
    build: "./worker/go"
    env: {
     "POSTGRES_USER": "secret://db-creds/username"
     "POSTGRES_PASSWORD": "secret://db-creds/password"
    }
    memory: 32Mi
  }
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
    memory: 128Mi
  }
  result: {
    labels: {
      component: "result"
    }
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
    memory: 128Mi
  }
  resultui: {
    labels: {
      component: "resultui"
    }
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
    if ! args.dev {
      memory: 32Mi
    }
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
</pre>
</details>

Note: you can find more information about memory constraints in [the official documentation](https://docs.acorn.io/reference/memory)

[Previous](./labels.md)  
[Next](./job.md)