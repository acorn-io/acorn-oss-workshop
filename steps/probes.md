Defining probes for each container is a best practice as it allows to make sure a container is running and ready to receive traffic.

Acorn offers a way to specify probes using the *probes* property in the definition of a container. *liveness*, *readiness* and *startup* are the 3 types of probes available:
- a *liveness* probe makes sure the container is up and running (it restarts the container if the probe fails)
- a *readiness* probe makes sure the container is ready to receive traffic
- a *startup* probe is used to give a container additional time to start 

In this step we will just add a *readiness* probe to the *voteui* but the same approach would apply to the others containers of the VotingApp.

Change the *voteui* container specifying a readiness probe as follows (this probe ensures *voteui* container is able to serve request on port 8080):

```
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
    probes: [
      {
        type: "readiness"
        initialDelaySeconds: 10
        periodSeconds: 5
        http: {
            url: "http://localhost:8080"
        }
      }
    ]
  }
```

Run the application using this new version of the Acornfile:

```
acorn run -n vote .
```

You should see that *voteui* never becomes ready:

```
...
[containers: voteui is not ready]
```

This is a normal behavior as we specified a wrong port number in the readiness probe definition (we used port *8080* whereas *voteui* container actually listens on port *80*). Let's fix that and update the probe definition of the *voteui* container so it looks like the following:

```
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
    probes: [
      {
        type: "readiness"
        initialDelaySeconds: 10
        periodSeconds: 5
        http: {
            url: "http://localhost"
        }
      }
    ]
  }
```

Stop the application and restart it with this corrected version of the specification:

```
acorn rm app --force --all
acorn run -n vote .
```

This time you should see the application is up and running after a couple of seconds.

In this step we've only added a readiness probe to *voteui* container to illustrate the way probes are defined. Feel free to add such probes to the other containers of the VotingApp.

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
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
    probes: [
      {
        type: "readiness"
        initialDelaySeconds: 10
        periodSeconds: 5
        http: {
            url: "http://localhost"
        }
      }
    ]
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
<pre>

</pre>
</details>

Note: you can find more information about probes in [the Acorn documentation](https://docs.acorn.io/authoring/containers#probes)

[Previous](./constraints.md)  
[Next](./job.md)