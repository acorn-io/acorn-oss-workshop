In the previous step we described the VotingApp in an Acornfile and ran it.  As a reminder, we ended up with the following Acornfile:

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
  }

  worker: {
    build: "./worker/go"
    env: {
     "POSTGRES_USER": "postgres"
     "POSTGRES_PASSWORD": "postgres"
    }
  }

  db: {
    image: "postgres:15.0-alpine3.16"
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

If you look at how the username and password are provided to the *db* container, and to the other containers that need to connect to it, you’ll notice this is not clean nor secure because those credentials are in plain text in each container.  

In this step we will improve the Acornfile using Acorn secrets.

## About Acorn secrets

To secure sensitive pieces of information we can use Acorn secrets, there are several types of secrets available:

- Basic secrets are used to generate and/or store usernames and passwords
- Token secrets are used to generate and/or store long secret strings
- Template secret are used to store configuration files that contain sensitive information
- a secret can be of type Opaque, it’s a  generic secret that can contain unstructured data
- a Generated secret is created from the output of a job (more on Job usage in a next article)

In the following we will use a secret of type Basic as this one handles simple username and password information.

## Adding a secrets in the Acornfile

First add a top level secrets key in your Acornfile as follows:

```
secrets: {
    "db-creds": {
        type: "basic"
        data: {
            username: "postgres"
            password: "postgres"
        }
    }
}
```

It defines a secret named *pg-creds*, of type basic, with the value *postgres* set for both the username and password properties.

Next, reference the secret’s username and password into the *worker*, *db* and *result* containers. To do so use the following syntax :

```
secrets://SECRET_NAME/PROPERTY_NAME
```

For instance, the new definition of the *db* container is as follows:

```
db: {
  image: "postgres:15.0-alpine3.16"
  ports: "5432/tcp"
  env: {
    "POSTGRES_USER": "secret://db-creds/username"
    "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
}
```

The same changes need be done for the *worker* and *result* containers:

```
worker: {
  build: "./worker/go"
  env: {
    "POSTGRES_USER": "secret://db-creds/username"
    "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
}
```

and to the *result* container:

```
result: {
  build: "./result"
  ports: "5000/http"
  env: {
    "POSTGRES_USER": "secret://db-creds/username"
    "POSTGRES_PASSWORD": "secret://db-creds/password"
  }
}
```

The Acornfile is slightly better now because the secret is defined once and then referenced in the containers which need it. But, the credentials are still in plain text in the definition of the secret, which is what we’d like to avoid.

Secrets of type Basic or Token have a special feature which allows the auto generation of values in case they are not provided. From the VotingApp perspective we need to define credentials for the *db* container and make sure both *worker* and *result* containers can connect to it, but we don’t need to have access to those credentials externally. In the definition of the secret we can then empty both username and password:

```
secrets: {
    "db-creds": {
        type: "basic"
        data: {
            username: ""
            password: ""
        }
    }
}
```

The new version of the Acornfile now looks as follows:

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
```

Run the app using this new version of the Acornfile:

```
acorn run -n vote .
```

Note: if the app was already running you can use the following command to update it:

```
acorn run -n vote --update .
```

After a couple of minutes you will get http endpoints (different from the ones you got in the previous step) to access both *voteui* and *resultui* containers:

- voteui : http://voteui-vote-c7bc34b6.jy7jy0.alpha.on-acorn.io

- resultui: http://resultui-vote-f1825499.jy7jy0.alpha.on-acorn.io

You can now access the Vote UI, select your favorite pet, then make sure your vote has been taken into account accessing the result UI.

<details>
  <summary markdown="span">If you curious about...</summary>

...what happened under the hood, you can see a new namespace has been created and the application is running inside this one. 

```
$ kubectl get ns
...
vote-7830ef54-bef    Active   61s   <-- new namespace created
```

Within this same namespace you could see the Kubernetes Deployment and Service as in the previous step, and you would also get the *db-creds* secret that is now part of the application: 

```
$ kubectl get secret -n vote-7830ef54-bef db-creds
NAME       TYPE                     DATA   AGE
db-creds   secrets.acorn.io/basic   2      58s
```
</details>

There are cases where we want to create a secret externally, this is what we will explore in the next part.

## Managing secrets from the command line

Using the following command check the list of actions that can be performed on secrets:

```
acorn secret --help
```

First create a secret named postgres-credential, of type basic, which contains 2 keys (username and password) and the associated values:

```
acorn secrets create --type basic --data username=postgres --data password=postgres postgres-credentials
```

Once created we can easily retrieve the value of the secret (in plain text !):

```
$ acorn secret reveal postgres_credentials
NAME                   TYPE      KEY        VALUE
postgres-credentials   basic     password   postgres
postgres-credentials   basic     username   postgres
```

Acorn added an encryption feature in the latest release thus making it possible to encrypt a secret. Let’s then encrypt the postgres-credentials secret:

```
acorn secret encrypt postgres-credentials
```

You will get the encrypted version similar to the following one (note the ACORNENC string at the beginning):

```
ACORNENC:eyJ6a0RmcUEzQlF4aUxybGpyWnJtd2l2dzhDeTdIOTlIS216b3NXcXR3WXg0IjoiLVlzYU9yOUJvbVQ2QVk3RFVsLVFMRE9pYmp5MTlsMW9UUFJGSDViVWQwd1kzSmxTbzhYcER2Z2pJZzZmQ1pHSWxlbUtiN1ZPeUxfaHBWYkI2NFRZMndFcHAyVSJ9
```

Behind the hood, the encryption is done using the cluster’s public key which can be obtained with the following command:

```
acorn info
```

This returns a result similar to that one, the encryption key being defined under the .namespace.publicKeys.keyID property:

```
---
client:
  cli:
    hubServers:
    - acorn.io
  version:
    commit: 7a4904c7d2d1077885ad6e2775550666b0a96905
    tag: v0.5.0
project:
  publicKeys:
  - keyID: zkDfqA3BQxiLrljrZrmwivw8Cy7H99HKmzosWqtwYx4
server:
  apiServerImage: ghcr.io/acorn-io/acorn:v0.5.0
  config:
    acornDNS: auto
    acornDNSEndpoint: https://alpha-dns.acrn.io/v1
    autoUpgradeInterval: 5m
    builderPerProject: false
    clusterDomains:
    - .jy7jy0.alpha.on-acorn.io
    defaultPublishMode: defined
    httpEndpointPattern: '{{hashConcat 8 .Container .App .Namespace | truncate}}.{{.ClusterDomain}}'
    ignoreUserLabelsAndAnnotations: false
    ingressClassName: null
    internalClusterDomain: svc.cluster.local
    internalRegistryPrefix: ""
    letsEncrypt: disabled
    letsEncryptEmail: ""
    letsEncryptTOSAgree: false
    podSecurityEnforceProfile: baseline
    publishBuilders: false
    recordBuilds: false
    setPodSecurityEnforceProfile: true
    useCustomCABundle: false
    workloadMemoryDefault: 0
    workloadMemoryMaximum: 0
  controllerImage: ghcr.io/acorn-io/acorn:v0.5.0
  dirty: false
  gitCommit: 7a4904c7d2d1077885ad6e2775550666b0a96905
  letsEncryptCertificate: disabled
  tag: v0.5.0
  userConfig:
    acornDNS: null
    acornDNSEndpoint: null
    autoUpgradeInterval: null
    builderPerProject: null
    clusterDomains: null
    defaultPublishMode: ""
    httpEndpointPattern: null
    ignoreUserLabelsAndAnnotations: null
    ingressClassName: null
    internalClusterDomain: ""
    internalRegistryPrefix: null
    letsEncrypt: null
    letsEncryptEmail: ""
    letsEncryptTOSAgree: null
    podSecurityEnforceProfile: ""
    publishBuilders: null
    recordBuilds: null
    setPodSecurityEnforceProfile: null
    useCustomCABundle: false
    workloadMemoryDefault: null
    workloadMemoryMaximum: null
  version: v0.5.0+7a4904c7
```

We can then provide the db credential through this encrypted secret to the *db*, *worker* and *result* containers. For that purpose we use the *-s* flag and provide the name of the secret we want to use followed by the name of the secret it should be bound to.

We can now run the application and make sure it uses the external postgres-credentials secret with the following command:

```
acorn run -n vote -s postgres-credentials:db-creds --update .
```

As we’ve done previously we could verify the app is working fine and then vote for our favorite pet.

<details>
  <summary markdown="span">Acornfile you should have at the end of this step...</summary>
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
```
</details>

Note: you can find more information about secrets in [the official documentation](https://docs.acorn.io/authoring/secrets)

Note: In this step we explained how to define and use a secret of type basic in the Acornfile and also how we can use a secret created from the command line. We focused on the connection to the *db* container but we could use the same approach to secure the connection to *redis*. In that case we would rather use a secret of type template because redis defines usernames and passwords as acls in a configuration file.

[Previous](./acornfile.md)  
[Next](./volumes.md)