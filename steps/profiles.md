Sometimes we need to have the application configured differently, for instance between different environments. Args and profiles are 2 Acorn concepts which are often used together to provide a dynamic configuration to an application so it can defined a different set of default for each environment.

- *args* defines arguments that can be modified at build or runtime by the user
- *profiles* specify default arguments for different contexts

Let's see this in action on the VotingApp.

First, change the Acornfile adding this section at the very beginning:

```
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
```

The change above defines a default value for the *replicas* value. It also ensures this default value is modified if the app is ran using a specific profile (*dev or *test* in this example).

Next modify the definition of the *voteui* container adding the *scale* property so it looks as follows, the number of *voteui* containers now depends on the value of the *replicas* arguments:

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
    scale: args.replicas
    memory: 128Mi
  }
```

You can now test the following actions:

- run the app without args nor profile information

make sure the following command uses the default value of the *replicas* args, it should create 3 *voteui* containers:

```
acorn run -n vote --update .
```

Note: you can verify the running containers with ```acorn all``` (which lists all the Acorn resources) or with ```acorn containers``` (which only lists containers)

```
$ acorn containers
NAME                             APP       IMAGE                                                                     STATE     RESTARTCOUNT   CREATED   MESSAGE
vote.db-54b8487bc5-jsvq7         vote      postgres:15.0-alpine3.16                                                  running   0              30s ago
vote.result-6f6dff8dd8-7gzkn     vote      sha256:805ece5530411a09e8e97e59eb9477355ef260c9d9a4204667ce2a364cddcc53   running   0              30s ago
vote.worker-57bcd65d56-ml9cl     vote      sha256:f97cb92018967769330da09bba644da395c176f549e1d11bf48da3a8927cb62e   running   0              30s ago
vote.redis-86c6745f86-hs667      vote      redis:7.0.5-alpine3.16                                                    running   0              31s ago
vote.resultui-64f48dd775-dm9ht   vote      sha256:a6f9230076947f698858df6793097f6b239f3eef108297e4e4d62e9b6f258d9a   running   0              31s ago
vote.vote-6f448d9c4b-45lsw       vote      sha256:73836e77179e1eb068d4aa54d58350c0bc2ebf32b45aab80997273278fbd2373   running   0              31s ago
vote.voteui-86bb4b699f-bw4z4     vote      sha256:069dcc1ab8846d2eb6e8b57d39a29f07607a477f74ecde66d0ef0e02289300d6   running   0              31s ago
vote.voteui-86bb4b699f-nm2n9     vote      sha256:069dcc1ab8846d2eb6e8b57d39a29f07607a477f74ecde66d0ef0e02289300d6   running   0              31s ago
vote.voteui-86bb4b699f-zshbf     vote      sha256:069dcc1ab8846d2eb6e8b57d39a29f07607a477f74ecde66d0ef0e02289300d6   running   0              31s ago
```

- running the app overwriting the default args

The following command will not use the default value of the *replicas* args but it will use the user supplied value instead. Make sure it creates 5 *voteui* containers:

```
acorn run -n vote --update . --replicas=5
```

- running the app specifying a profile

The following command will not use the default value of the *replicas* args but it will use the value defined in the *test* profile instead. Make sure it creates 2 containers for the *voteui* microservice:

```
acorn run -n vote --update --profile test .
```

We only used the *voteui* container to illustrate the usage of args / profile but we could have used other stateless containers in the same way. Also, we only specified a simple args (a string) but more complex structure could be used as well.

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
<pre>
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
      if !args.dev {
        "/var/lib/postgresql/data": "volume://db"
      }
    }
    memory: 128Mi
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
    memory: 128Mi
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

Note: you can find more information about Arguments and Profiles in [the Acorn documentation](https://docs.acorn.io/authoring/args-and-profiles)

[Previous](./development_mode.md)  
[Next](./labels.md)