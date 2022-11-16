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

To represent the microservices of the VotingApp, we create an Acornfile in the *votingapp* folder. This file only contains the containers top level key and an empty element for each microservice:

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

As the microservice will run in containers, we need to specify how the containers can be built or which image it is based on:

- for the vote-ui, vote, worker, result and result-ui microservices we use the build.context property to reference the location of the Dockerfile that will be used to build the image
- for db and redis we specify the image property

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

For the postgres image to run we need to provide it the POSTGRES_PASSWORD environment variable. In this example we also define the POSTGRES_USER. 

The definition of the db container is as follows:

```
db: {
   image: "postgres:13.2-alpine"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
}
```

As result needs to connect to db, we specify the credentials in that container too:

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

worker also communicates with db so we also give it the credentials it needs:

```
worker: {
   build: "./worker/go"
   env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
   }
 }
```

In a future article in the series we’ll show how to use Acorn secrets to have a much cleaner / safer approach and avoid to specify the password in the plain text :)

In order for the container of the application to communicate with each other we need to define the network ports for each one. As defined in the documentation, there are 3 scopes to specify the ports:

- internal allows communication between containers within the same Acorn app
- expose allows communication between containers within the cluster
- publish allows containers to be reached from outside of the cluster

As vote, result, redis and db microservices only need to be reachable from other containers within the same application, we use the default internal scope for each of them.

As vote-ui and result-ui need to be reachable from the outside world we use the publish scope for both of them.

The Acornfile now looks as follows:

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

We now have a first (minimal) version of the Acornfile which specifies the application. Let’s make sure the Voting App can be run using that one.

## Testing the application

We run the Voting Application using the acorn CLI:

```
acorn run -n vote .
```

It takes a few tens of seconds for the application to be up and running. When it’s ready we are provided http endpoints on the acorn.io domain for both vote-ui and result-ui containers:

vote-ui : http://resultui-vote-5ccf7969.po4pog.alpha.on-acorn.io

result-ui: http://voteui-vote-5ccf7969.po4pog.alpha.on-acorn.io

Using the acorn CLI we can visualize all the acorn resources created:

```
% acorn all
 
APPS:
NAME      IMAGE          HEALTHY   UP-TO-DATE   CREATED   ENDPOINTS                                                                                                                                  MESSAGE
vote      e48599ee3ff2   7         7            20m ago   http://resultui-vote-5ccf7969.po4pog.alpha.on-acorn.io => resultui:80, http://voteui-vote-5ccf7969.po4pog.alpha.on-acorn.io => voteui:80   OK
 
CONTAINERS:
NAME                             APP       IMAGE                                                                     STATE     RESTARTCOUNT   CREATED   MESSAGE
vote.db-6dcdc586fc-9v6m2         vote      postgres:15.0-alpine3.16                                                  running   0              20m ago
vote.redis-76f88c775-67l9j       vote      redis:7.0.5-alpine3.16                                                    running   0              20m ago
vote.result-5c467bbf6c-hz7s9     vote      sha256:ca344fce985f8218a8dbdcda436dadb6ac42abb596c906be5504176bef2a3903   running   0              20m ago
vote.resultui-54f8b6c865-bfw7h   vote      sha256:bd93fe432c1322b331c5b23057513bbdc01be5e3373eee617a31f4a507feeb7c   running   0              20m ago
vote.vote-db445dfcc-bhxzd        vote      sha256:8d027e0823d9fb091d9e27ba8b8e01d0b7bfffc47158e08ea8a7d056ca537be3   running   0              20m ago
vote.voteui-6cf4bb8b7d-5t4jr     vote      sha256:872361865af0f935cf45267e0b615bf7c9acfec87391bdac172690a1dde81bae   running   0              20m ago
vote.worker-6446d5cfff-x8c4v     vote      sha256:e3722b63712a85259ee3868620d45fb2266ed13d44149e06bf5015bbc35c027e   running   0              20m ago
 
VOLUMES:
NAME      APP-NAME   BOUND-VOLUME   CAPACITY   STATUS    ACCESS-MODES   CREATED
 
SECRETS:
ALIAS     NAME      TYPE      KEYS      CREATED
```

The application’s containers have been created and exposed. Currently there are no secrets nor volumes as we did not defined those top level elements in the Acornfile.

If you are curious about what happened under the hood, we could see that a new Kubernetes namespace has been created in the cluster, this one is dedicated to our newly created acorn application:

```
$ kubectl get ns
NAME                STATUS   AGE
default             Active   4h48m
kube-system         Active   4h48m
kube-public         Active   4h48m
kube-node-lease     Active   4h48m
traefik             Active   4h8m
acorn               Active   3h56m
acorn-system        Active   3h56m
vote-5ccf7969-b2f   Active   34m     <strong><- namespace created for the application</strong>
Within this namespace there are a Deployment / Pod and a Service for each microservice of the Voting App:

$ kubectl get all -n vote-5ccf7969-b2f
NAME                            READY   STATUS    RESTARTS   AGE
pod/worker-6446d5cfff-x8c4v     1/1     Running   0          27m
pod/voteui-6cf4bb8b7d-5t4jr     1/1     Running   0          27m
pod/redis-76f88c775-67l9j       1/1     Running   0          27m
pod/vote-db445dfcc-bhxzd        1/1     Running   0          27m
pod/result-5c467bbf6c-hz7s9     1/1     Running   0          27m
pod/resultui-54f8b6c865-bfw7h   1/1     Running   0          27m
pod/db-6dcdc586fc-9v6m2         1/1     Running   0          27m
 
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db         ClusterIP   10.43.150.236   <none>        5432/TCP   27m
service/redis      ClusterIP   10.43.242.30    <none>        6379/TCP   27m
service/result     ClusterIP   10.43.224.90    <none>        5000/TCP   27m
service/resultui   ClusterIP   10.43.72.204    <none>        80/TCP     27m
service/vote       ClusterIP   10.43.239.217   <none>        5000/TCP   27m
service/voteui     ClusterIP   10.43.81.237    <none>        80/TCP     27m
 
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/worker     1/1     1            1           27m
deployment.apps/voteui     1/1     1            1           27m
deployment.apps/redis      1/1     1            1           27m
deployment.apps/vote       1/1     1            1           27m
deployment.apps/result     1/1     1            1           27m
deployment.apps/resultui   1/1     1            1           27m
deployment.apps/db         1/1     1            1           27m
 
NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/worker-6446d5cfff     1         1         1       27m
replicaset.apps/voteui-6cf4bb8b7d     1         1         1       27m
replicaset.apps/redis-76f88c775       1         1         1       27m
replicaset.apps/vote-db445dfcc        1         1         1       27m
replicaset.apps/result-5c467bbf6c     1         1         1       27m
replicaset.apps/resultui-54f8b6c865   1         1         1       27m
replicaset.apps/db-6dcdc586fc         1         1         1       27m
On top of that, an Ingress resource has been created so the web interfaces (vote-ui and result-ui) can be exposed through the cluster’s Ingress Controller (Traefik in our setup):

$ kubectl get ingress -n vote-5ccf7969-b2f
NAME       CLASS    HOSTS                                             ADDRESS         PORTS   AGE
voteui     <none>   voteui-vote-5ccf7969.po4pog.alpha.on-acorn.io     74.220.24.195   80      31m
resultui   <none>   resultui-vote-5ccf7969.po4pog.alpha.on-acorn.io   74.220.24.195   80      31m
We can then remove the application using the following command:

acorn rm vote
```
