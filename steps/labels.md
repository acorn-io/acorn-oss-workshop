It is often usefull to add labels on application's resources. Labels allow to group resources in a logical way and also to filter them. As you will see below, Acorn makes it very easy to add labels on the whole application or on indivual pieces.

First modify the Acornfile adding the *labels* top-level key at the top of the file:

```
labels: {
    application: "votingapp"
}
```

Next modify the definition of each container adding a new label property with *component* as the key and the container's name as the associated value. The following shows the changes done on the *voteui* container:

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
    memory: 128Mi
  }
```

Once you've added the new label for each container, update the application and check the labels now existing on the application's resources.

```
acorn run -n vote --update  .
```

<details>
  <summary markdown="span">If you curious about...</summary>

...what happened under the hood, you can see that the Pods created now have 2 additional labels:

- application
- component

Those labels were added on top of the labels automatically set when running the acorn application:

- acorn.io/app-name
- acorn.io/app-namespace
- acorn.io/container-name
- acorn.io/managed
- port-number.acorn.io/xxx
- service-name.acorn.io/yyy

To verify this, first get the Kubernetes namespace created for that Acorn app (your namespace will be different):

```
$ kubectl get ns
...
vote-79e1c2f0-c77    Active   6m
```

Next check the labels of the Pods in that namespace:

```
kubectl get po --show-labels -n vote-79e1c2f0-c77
```
</details>

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
      dirs: {
        "/usr/share/nginx/html": "./vote-ui"
      }
    }
    build: {
      context: "./vote-ui"
    }
    ports: publish : "80/http"
    scale: args.replicas
    memory: 128Mi
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
    memory: 128Mi
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
    memory: 128Mi
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
      "PGDATA": "/var/lib/postgresql/data/db"
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
    memory: std.ifelse(args.dev, 1Gi, 128Mi)
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

Note: you can find more information about labels in [the Acorn documentation](https://docs.acorn.io/authoring/labels)

[Previous](./profiles.md)  
[Next](./probes.md)