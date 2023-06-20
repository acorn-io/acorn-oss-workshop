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
- services: cloud services that will be provisioned for an application

To represent the microservices of the VotingApp, create an Acornfile in the *votingapp* folder.  
This file should only contain the *containers* top level key and an empty element for each microservice as follows:

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

- voteui : http://voteui-vote-c7bc34b6.xojtjt.oss-acorn.io 

- resultui: http://resultui-vote-f1825499.xojtjt.oss-acorn.io

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
NAME      IMAGE          HEALTHY   UP-TO-DATE   CREATED     ENDPOINTS                                                                                                                        MESSAGE
vote      603de98b9a63   7         7            8m21s ago   http://resultui-vote-f1825499.xojtjt.oss-acorn.io => resultui:80, http://voteui-vote-c7bc34b6.xojtjt.oss-acorn.io => voteui:80   OK

CONTAINERS:
NAME                             APP       IMAGE                                                                     STATE     RESTARTCOUNT   CREATED     MESSAGE
vote.db-dd4d6646f-2gj2w          vote      postgres:13.2-alpine                                                      running   0              8m21s ago   
vote.redis-949fd9cd7-6tvnn       vote      redis:6.2-alpine3.13                                                      running   0              8m21s ago   
vote.result-69ccdc57b5-xjd4t     vote      sha256:4ba7605be0e824efc1ef1f18f0aa6a93c312cebf717b922703c3c5a85ed137e1   running   0              8m21s ago   
vote.resultui-74fb54c895-2cf26   vote      sha256:57ddc81d1659d6e58defd0b68f1734f8f9f9104f128379dfd53bbeb1b70b1a79   running   0              8m21s ago   
vote.vote-6c54747c6d-qgjds       vote      sha256:7c0dd21bf7542ed63fc96b119ca97adb48c90e66492fe5db332719552e68215b   running   0              8m21s ago   
vote.voteui-7fc59756d9-r9s4j     vote      sha256:7c565686dc24c53450033bfeb0fea6db85fc1ea59b67f4007a9dd3591ac22370   running   0              8m21s ago   
vote.worker-bdb8454b7-4mfbc      vote      sha256:e0a5a7e5b1d0b4e25a0f4a8ed51934223ca570f9b78b43668e1239e2ca33925b   running   0              8m21s ago   

VOLUMES:
NAME      APP-NAME   BOUND-VOLUME   CAPACITY   VOLUME-CLASS   STATUS    ACCESS-MODES   CREATED

SECRETS:
NAME      TYPE      KEYS      CREATED
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
vote-81255b28-ad4   Active   8m21s <- namespace created for the application
```

Within this namespace there are a Deployment / Pod and a Service for each microservice of the Voting App:

```
$ kubectl get all -n vote-81255b28-ad4
NAME                           READY   STATUS    RESTARTS   AGE
pod/voteui-f77bf48b6-v22kx     1/1     Running   0          8m21s
pod/redis-6564f9d99-mfdc4      1/1     Running   0          8m21s
pod/resultui-9f6546859-f9csq   1/1     Running   0          8m21s
pod/vote-b4b7dc9b9-5vftx       1/1     Running   0          8m21s
pod/worker-6fd688c9cf-vf2jc    1/1     Running   0          8m21s
pod/result-594794bbfc-xqgkp    1/1     Running   0          8m21s
pod/db-754bb4c9bf-4cv7r        1/1     Running   0          8m21s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db         ClusterIP   10.43.54.204    <none>        5432/TCP   8m21s
service/redis      ClusterIP   10.43.53.154    <none>        6379/TCP   8m21s
service/result     ClusterIP   10.43.49.253    <none>        5000/TCP   8m21s
service/resultui   ClusterIP   10.43.191.30    <none>        80/TCP     8m21s
service/vote       ClusterIP   10.43.203.229   <none>        5000/TCP   8m21s
service/voteui     ClusterIP   10.43.199.118   <none>        80/TCP     8m21s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/voteui     1/1     1            1           8m21s
deployment.apps/redis      1/1     1            1           8m21s
deployment.apps/resultui   1/1     1            1           8m21s
deployment.apps/vote       1/1     1            1           8m21s
deployment.apps/worker     1/1     1            1           8m21s
deployment.apps/result     1/1     1            1           8m21s
deployment.apps/db         1/1     1            1           8m21s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/voteui-f77bf48b6     1         1         1       8m21s
replicaset.apps/redis-6564f9d99      1         1         1       8m21s
replicaset.apps/resultui-9f6546859   1         1         1       8m21s
replicaset.apps/vote-b4b7dc9b9       1         1         1       8m21s
replicaset.apps/worker-6fd688c9cf    1         1         1       8m21s
replicaset.apps/result-594794bbfc    1         1         1       8m21s
replicaset.apps/db-754bb4c9bf        1         1         1       8m21s
```

On top of that, an Ingress resource has been created so the web interfaces (*voteui* and *resultui*) can be exposed through the cluster’s Ingress Controller (Traefik in our setup):

```
$ kubectl get ingress -n vote-81255b28-ad4
NAME       CLASS     HOSTS                                               ADDRESS          PORTS   AGE
voteui     traefik   http://voteui-vote-c7bc34b6.xojtjt.oss-acorn.io     192.168.205.2    80      8m21s
resultui   traefik   http://resultui-vote-f1825499.xojtjt.oss-acorn.io   192.168.205.2    80      8m21s
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
[Next](./ops.md)
