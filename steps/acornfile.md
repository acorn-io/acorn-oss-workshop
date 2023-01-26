## Defining the application in an Acornfile

As described in the documentation, an Acornfile contains the following top level elements:

- args: defines arguments the consumer can provide
- profiles: defines a set of default arguments for different deployment types
- containers: defines the containers to run the application
- volumes: defines persistent storage volumes for the containers to consume
- jobs: defines tasks to run on changes or via cron
- acorns: other Acorn applications that need to be deployed with your app
- secrets: defines secret bits of data that are automatically generated or passed by the user
- localData: default data and configuration variables
- routers: support path based HTTP routing to expose multiple containers through a single published service

To represent the microservices of the VotingApp, create an Acornfile in the *votingapp* folder. This file should only contain the *containers* top level key and an empty element for each microservice as follows:

```
containers: {
  voteui: {
  }
 
  vote: {
  }
 
  redis: {
  }
 
  worker: {
  }
 
  db: {
  }
 
  result: {
  }
 
  resultui: {
  }
}
```

As the microservice will run in containers, we need to specify either how the containers can be built or the image it is based on:

- for *voteui*, *vote*, *worker*, *result* and *resultui* microservices we use the *build.context* property to reference the location of the Dockerfile which will be used to build the image
- for *db* and *redis* we specify the image property

Make sure the Acornfile now looks as follows:

```
containers: {
 voteui: {
   build: {
     context: "./vote-ui"
   }
 }
 
 vote: {
   build: {
     context: "./vote"
   }
 }
 
 redis: {
   image: "redis:6.2-alpine3.13"
 }
 
 worker: {
   build: {
     context: "./worker/go"
   }
 }
 
 db: {
   image: "postgres:13.2-alpine"
 }
 
 result: {
   build: {
     context: "./result"
   }
 }
 
 resultui: {
   build: {
     context: "./result-ui"
   }
 }
}
```

For the postgres image to run we need to provide it the *POSTGRES_PASSWORD* environment variable. In this example we also define the *POSTGRES_USER*. 

The definition of the *db* container is as follows:

```
db: {
   image: "postgres:13.2-alpine"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
}
```

As *result* needs to connect to *db*, we specify the credentials in that container too:

```
result: {
   build: "./result"
   ports: "5000/http"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
}
```

as *worker* also communicates with *db* we give it the credentials it needs:

```
worker: {
   build: "./worker/go"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
 }
```

Note: in the next step we’ll show how to use Acorn secrets to avoid to specify the password in plain text

In order for the containers of the application to communicate with each others we need to define the network ports for each one. As defined in the documentation, there are 3 scopes to specify the ports:

- **internal** allows communication between containers within the same Acorn app
- **expose** allows communication between containers within the cluster
- **publish** allows containers to be reached from outside of the cluster

As *vote*, *result*, *redis* and *db* microservices only need to be reachable from other containers within the same application, we use the **internal** scope (default one) for each of them.

As *voteui* and *resultui* need to be reachable from the outside world we use the **publish** scope for both of them.

Make sure your Acornfile now looks as follows:

```
containers: {
 voteui: {
   build: "./vote-ui"
   ports: publish : "80/http"
 }
 
 vote: {
   build: "./vote"
   ports: "5000/tcp"
 }
 
 redis: {
   image: "redis:6.2-alpine3.13"
   ports: "6379/tcp"
 }
 
 worker: {
   build: "./worker/go"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
 }
 
 db: {
   image: "postgres:13.2-alpine"
   ports: "5432/tcp"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
 }
 
result: {
   build: "./result"
   ports: "5000/http"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
 }
 
 resultui: {
   build: "./result-ui"
   ports: publish : "80/http"
 }
}
```

You now have a first (minimal) version of the Acornfile which specifies the application. We will now use it to build and run the VotingApp.

## Testing the application

Use the following command to run the application:

```
acorn run -n vote .
```

It will take a couple of minutes for the application to be up and running (all the containers need to be built first). When it’s ready you should be returned the http endpoints (on the *acorn.io* domain) to access both *voteui* and *resultui* containers.

Your endpoints should have the same format as the following ones (the identifiers will be different though):

- voteui : http://voteui-vote-c7bc34b6.jy7jy0.alpha.on-acorn.io

- resultui: http://resultui-vote-f1825499.jy7jy0.alpha.on-acorn.io

You can now access the Vote UI, select your favorite pet, then make sure your vote has been taken into account accessing the result UI.

![Vote UI](./images/acornfile/vote-ui.png)

![Result UI](./images/acornfile/result-ui.png)

Using the following command you can visualize all the acorn resources created:

```
acorn all
```

It should return a result similar to the following one:

```
APPS:
NAME      IMAGE          HEALTHY   UP-TO-DATE   CREATED     ENDPOINTS                                                                                                                                  MESSAGE
vote      3783c2ae5834   7         7            8m21s ago   http://resultui-vote-f1825499.jy7jy0.alpha.on-acorn.io => resultui:80, http://voteui-vote-c7bc34b6.jy7jy0.alpha.on-acorn.io => voteui:80   OK

CONTAINERS:
NAME                            APP       IMAGE                                                                     STATE     RESTARTCOUNT   CREATED     MESSAGE
vote.result-594794bbfc-xqgkp    vote      sha256:18ec679d38058d97d5e0d7b101a4fe3ad01f44c228bd3abcb8f36961b8cb9d76   running   0              8m20s ago
vote.resultui-9f6546859-f9csq   vote      sha256:a6f9230076947f698858df6793097f6b239f3eef108297e4e4d62e9b6f258d9a   running   0              8m20s ago
vote.worker-6fd688c9cf-vf2jc    vote      sha256:f97cb92018967769330da09bba644da395c176f549e1d11bf48da3a8927cb62e   running   0              8m20s ago
vote.db-754bb4c9bf-4cv7r        vote      postgres:13.2-alpine                                                      running   0              8m21s ago
vote.redis-6564f9d99-mfdc4      vote      redis:6.2-alpine3.13                                                      running   0              8m21s ago
vote.vote-b4b7dc9b9-5vftx       vote      sha256:73836e77179e1eb068d4aa54d58350c0bc2ebf32b45aab80997273278fbd2373   running   0              8m21s ago
vote.voteui-f77bf48b6-v22kx     vote      sha256:069dcc1ab8846d2eb6e8b57d39a29f07607a477f74ecde66d0ef0e02289300d6   running   0              8m21s ago

VOLUMES:
NAME      APP-NAME   BOUND-VOLUME   CAPACITY   STATUS    ACCESS-MODES   CREATED

SECRETS:
ALIAS     NAME      TYPE      KEYS      CREATED
```

The application’s containers have been created and exposed. Currently there are no secrets nor volumes as we did not define those top level elements in the Acornfile (yet).

<details>
  <summary markdown="span">If you are curious about...</summary>

...what happened under the hood, we could see that a new Kubernetes namespace has been created in the cluster, this one is dedicated to our newly created acorn application:

```
$ kubectl get ns
NAME                STATUS   AGE
default             Active   20m
kube-system         Active   20m
kube-public         Active   20m
kube-node-lease     Active   20m
acorn               Active   19m
acorn-system        Active   19m
acorn-image-system  Active   19m
vote-81255b28-ad4   Active   2m10s <- namespace created for the application
```

Within this namespace there are a Deployment / Pod and a Service for each microservice of the Voting App:

```
$ kubectl get all -n vote-81255b28-ad4
NAME                           READY   STATUS    RESTARTS   AGE
pod/voteui-f77bf48b6-v22kx     1/1     Running   0          2m53s
pod/redis-6564f9d99-mfdc4      1/1     Running   0          2m53s
pod/resultui-9f6546859-f9csq   1/1     Running   0          2m53s
pod/vote-b4b7dc9b9-5vftx       1/1     Running   0          2m53s
pod/worker-6fd688c9cf-vf2jc    1/1     Running   0          2m53s
pod/result-594794bbfc-xqgkp    1/1     Running   0          2m53s
pod/db-754bb4c9bf-4cv7r        1/1     Running   0          2m53s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db         ClusterIP   10.43.54.204    <none>        5432/TCP   2m53s
service/redis      ClusterIP   10.43.53.154    <none>        6379/TCP   2m53s
service/result     ClusterIP   10.43.49.253    <none>        5000/TCP   2m53s
service/resultui   ClusterIP   10.43.191.30    <none>        80/TCP     2m53s
service/vote       ClusterIP   10.43.203.229   <none>        5000/TCP   2m53s
service/voteui     ClusterIP   10.43.199.118   <none>        80/TCP     2m53s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/voteui     1/1     1            1           2m53s
deployment.apps/redis      1/1     1            1           2m53s
deployment.apps/resultui   1/1     1            1           2m53s
deployment.apps/vote       1/1     1            1           2m53s
deployment.apps/worker     1/1     1            1           2m53s
deployment.apps/result     1/1     1            1           2m53s
deployment.apps/db         1/1     1            1           2m53s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/voteui-f77bf48b6     1         1         1       2m53s
replicaset.apps/redis-6564f9d99      1         1         1       2m53s
replicaset.apps/resultui-9f6546859   1         1         1       2m53s
replicaset.apps/vote-b4b7dc9b9       1         1         1       2m53s
replicaset.apps/worker-6fd688c9cf    1         1         1       2m53s
replicaset.apps/result-594794bbfc    1         1         1       2m53s
replicaset.apps/db-754bb4c9bf        1         1         1       2m53s
```

On top of that, an Ingress resource has been created so the web interfaces (*voteui* and *resultui*) can be exposed through the cluster’s Ingress Controller (Traefik in our setup):

```
$ kubectl get ingress -n vote-81255b28-ad4
NAME       CLASS     HOSTS                                             ADDRESS          PORTS   AGE
voteui     traefik   voteui-vote-c7bc34b6.jy7jy0.alpha.on-acorn.io     89.145.160.110   80      3m41s
resultui   traefik   resultui-vote-f1825499.jy7jy0.alpha.on-acorn.io   89.145.160.110   80      3m41s
```
</details>

You can then remove the application:

```
acorn rm vote
```

Wait a couple of seconds and make sure the list of acorn resources is now empty:

```
acorn all
```

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
<pre>
containers: {
  voteui: {
    build: "./vote-ui"
    ports: publish : "80/http"
  }
  vote: {
    build: "./vote"
    ports: "5000/tcp"
  }
  redis: {
    image: "redis:6.2-alpine3.13"
    ports: "6379/tcp"
  }
  worker: {
    build: "./worker/go"
    env: {
      "POSTGRES_USER": "postgres"
      "POSTGRES_PASSWORD": "postgres"
    }
  }
  db: {
    image: "postgres:13.2-alpine"
    ports: "5432/tcp"
    env: {
      "POSTGRES_USER": "postgres"
      "POSTGRES_PASSWORD": "postgres"
    }
  }
  result: {
    build: "./result"
    ports: "5000/http"
    env: {
      "POSTGRES_USER": "postgres"
      "POSTGRES_PASSWORD": "postgres"
    }
  }
  resultui: {
    build: "./result-ui"
    ports: publish : "80/http"
  }
}
</pre>
</details>

Note: you can find more information about Acornfile in [Authoring Acornfile](https://docs.acorn.io/authoring)

[Previous](./votingapp.md)  
[Next](./secret.md)
