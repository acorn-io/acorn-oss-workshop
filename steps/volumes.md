In the previous step we enhanced the Acornfile of the VotingApp and added Acorn secrets inside of it. In this step we will add a storage definition for both *redis* and *db* containers so we can persist their data. 

Both *db* and *redis* containers are databases but as we didn’t specify any storage related properties, the data of each container are stored in their own file system. This is usually fine in a development environment but definitely not what we want in production. 

In order to ensure data is persisted for both containers we will use Acorn volumes.

# Defining volumes in the Acornfile

Volumes is another top level key we can use in an Acornfile. It allows to define storage that will be referenced by the containers which need it. For our needs we define 2 different volumes, one for each database of the VotingApp. The following shows how those volumes are defined using the default StorageClass existing in the cluster (as we are using k3s a StorageClass based on LocalPath provisioner already exists in the cluster):

```
volumes: {
    "db": {
        size: 100M
    }
    "redis": {
        size: 100M
    }
}
```

Note: by default a volume is created using the *default* StorageClass, a size of *10G* and a "readWriteOnce" access mode. This can be customized using dedicated properties. 

Once defined in the *volumes* top level key, we need to mount each volumes in the corresponding container knowning that:
- postgres persists its data in */var/lib/postgresql/data*
- redis persists its data in */data*

The *db* container could then be changed as follows:

```
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
  }
```

Note: we specify the env var *PGDATA* as this can prevent error depending on the way the storage solution provides the volumes

the *redis* one as follows:

```
  redis: {
    image: "redis:7.0.5-alpine3.16"
    ports: "6379/tcp"
    dirs: {
      "/data": "volume://redis"
    }
  }
```

Update the application using this new version of the Acornfile:

```
acorn run -n vote --update .
```

As in the previous step, you should be returned the http endpoints used to access both *voteui* and *resultui* frontends. 

<details>
  <summary markdown="span">If you curious about...</summary>

... what happened under the hood, you can see 2 PersistentVolume have been created, one for each database container:

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pvc-dc2c7d70-de12-4e31-8a21-7c96005c432e   10G        RWO            Retain           Bound    vote-7830ef54-bef/redis   local-path              27s
pvc-54c9a948-c736-4792-905d-1ed47792f518   10G        RWO            Retain           Bound    vote-7830ef54-bef/db      local-path              27s
```

</details>

Note: from the cli it's possible to specify the caracteristics of a volume already defined in the Acornfile. For instance, to run the application you could have run the following command specifying 200M of storage for each volume (instead of the 100M specified in the Acornfile):

```
acorn run -n vote -v db,size=200M -v redis,size=200M --update .
```

Once you are done, you can remove the application and all the related resources:

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
     "PGDATA": "/var/lib/postgresql/data/db"
    }
    dirs: {
      "/var/lib/postgresql/data": "volume://db"
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

Note: you can find more information about volumes in [the Acorn documentation](https://docs.acorn.io/running/volumes)

[Previous](./secret.md)  
[Next](./constraints.md)