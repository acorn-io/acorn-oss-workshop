In the previous step we described the Voting App in an Acornfile and ran it. As a reminder, we ended up with the following Acornfile:

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

If you look at how the username and password are provided to the db container, and to the other containers that need to connect to it, you’ll notice this is not clean nor secure because those credentials are in plain text in each container. In this step we will improve the Acornfile using Acorn secrets.

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

Next, reference the secret’s username and password into the worker, db and result containers. To do so use the following syntax :

```
secrets://SECRET_NAME/PROPERTY_NAME
```

For instance, the new definition of the db container is as follows (the same changes need be done for the worker and result containers):

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

The Acornfile is slightly better now because the secret is defined once and then referenced in the containers which need it. But, the credentials are still in plain text in the definition of the secret, which is what we’d like to avoid.

Secrets of type Basic or Token have a special feature which allows the auto generation of values in case they are not provided. From the VotingApp perspective we need to define credentials for the db container and make sure both worker and result containers can connect to it, but we don’t need to have access to those credentials externally. In the definition of the secret we can then empty both username and password:

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

After a couple of minutes you will get http endpoints (different from the ones you got in the previous step) to access both vote-ui and result-ui interfaces.

- vote-ui : http://voteui-vote-df018e5a.tcc3t3.alpha.on-acorn.io

- result-ui: http://resultui-vote-df018e5a.tcc3t3.alpha.on-acorn.io

You can now access the Vote UI, select your favorite pet, then make sure your vote has been taken into account accessing the result UI.

<details>
  <summary markdown="span">If you curious about...</summary>

...what happened under the hood, you can see a new namespace has been created and the application is running inside this one. 

```
$ kubectl get ns
NAME                STATUS   AGE
default             Active   53m
kube-system         Active   53m
kube-public         Active   53m
kube-node-lease     Active   53m
acorn               Active   52m
acorn-system        Active   52m
vote-b74ab112-b0d   Active   3m12s <- new namespace
```

Within this same namespace you could see the Kubernetes Deployment and Service as in the previous step, and you would also get the *db-creds* secret that is now part of the application: 

```
$ kubectl get secret -n vote-b74ab112-b0d db-creds
NAME                         TYPE                             DATA   AGE
db-creds                     secrets.acorn.io/basic           2      3m56s
```
</details>

You can now remove the application and the associated secret:

```
acorn rm vote
acorn rm -s db-creds
```

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
$ acorn secret expose postgres_credentials
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
ACORNENC:eyJfb3VnbHVfOG00ZmRtR0hSQlh3QUJ6NU9ZcVpzOEdLS3ZiRXFCcmdjenc4IjoicFpNWC1BMXJ3bVFXT29iUFdRdXVWRmRsQVZ3N0tzSFRFMjdHRERKNUlIX3ZaaU9fU1hIQWRyMUtVY09CQ2kzTU9XWXJpN1JuOVR4a1FCWDZFaEVrSnBIUHdxWSJ9
```

Behind the hood, the encryption is done using the cluster’s public key which can be obtained with the following command:

```
acorn info
```

This returns a result similar to that one, the encryption key being defined under the .namespace.publicKeys.keyID property:

```
---
client:
  version:
    commit: f717d41a2cce4e37258bd1a3cb39dfd5841d4253
    tag: v0.4.0
namespace:
  publicKeys:
  - keyID: sgVNRZGN3QNnwHXslyNxyxZlytsP3yOk7SNDgWWnxjg
server:
  apiServerImage: ghcr.io/acorn-io/acorn:v0.4.0
  config:
    acornDNS: auto
    acornDNSEndpoint: https://alpha-dns.acrn.io/v1
    autoUpgradeInterval: 5m
    clusterDomains:
    - .klkue5.alpha.on-acorn.io
    defaultPublishMode: defined
    ingressClassName: null
    internalClusterDomain: svc.cluster.local
    letsEncrypt: disabled
    letsEncryptEmail: ""
    letsEncryptTOSAgree: false
    podSecurityEnforceProfile: baseline
    setPodSecurityEnforceProfile: true
  controllerImage: ghcr.io/acorn-io/acorn:v0.4.0
  dirty: false
  gitCommit: f717d41a2cce4e37258bd1a3cb39dfd5841d4253
  letsEncryptCertificate: disabled
  tag: v0.4.0
  userConfig:
    acornDNS: null
    acornDNSEndpoint: null
    autoUpgradeInterval: null
    clusterDomains: null
    defaultPublishMode: ""
    ingressClassName: null
    internalClusterDomain: ""
    letsEncrypt: null
    letsEncryptEmail: ""
    letsEncryptTOSAgree: null
    podSecurityEnforceProfile: ""
    setPodSecurityEnforceProfile: null
  version: v0.4.0+f717d41a
```

We can then provide the db credential through this encrypted secret to the db, worker and result containers. For that purpose we use the *-s* flag and provide the name of the secret we want to use followed by the name of the secret it should be bound to.

We can now run the application and make sure it uses the external postgres-credentials secret with the following command:

```
acorn run -n vote -s postgres-credentials:db-creds .
```

As we’ve done previously we could verify the app is working fine and then vote for our favorite pet.

Note: you can find more information about secrets in [the official documentation](https://docs.acorn.io/authoring/secrets)

## Summary

In this section we explained how to define and use a secret of type basic in the Acornfile and also how we can use a secrets created from the command line instead. We focused on the connection to the db container but we could use the same approach to secure the connection to redis. In that case we would rather use a secret of type template because redis defines usernames and passwords as acls in a configuration file.

[Previous](./acornfile.md)  
[Next](./volumes.md)