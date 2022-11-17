In the previous step we described the Voting App in an Acornfile and ran it in a demo cluster. As a reminder, we ended up with the following Acornfile:

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

If you look at how the username and password are provided to the db container, and to the other containers that need to connect to it, you’ll notice this is not clean nor secure because those credentials are in plain text in each container. In this post we will see how to improve the Acornfile by making it more secure with Acorn secrets.

## About Acorn secrets

To secure sensitive pieces of information we can use Acorn secrets, they are several types of secrets available:

- Basic secrets are used to generate and/or store usernames and passwords
- Token secrets are used to generate and/or store long secret strings
- Template secret are used to store configuration files that contain sensitive information
- a secret can be of type Opaque, it’s a  generic secret that can contain unstructured data
- a Generated secret is created from the output of a job (more on Job usage in a next article)

In the following we will use a secret of type Basic as this one handles simple username and password information.

Feel free to browse the documentation to get more information regarding the different types of secrets and how to use each of them.

## Adding a secrets in the Acornfile

First we can add a top level secrets key in our Acornfile as follows. It defines a secret named pg-creds, of type basic, with the value postgres set for both the username and password properties.

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

Next, we reference the secret’s username and password into the worker, db and result containers. To do so we use the following syntax :

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

We can run the app using this Acornfile:

```
acorn run -n vote .
```

After a couple of seconds you will get the http endpoints to access both vote-ui and result-ui interfaces, similar to the following ones:

```

```

You can now vote for your favorite pet once again.

SCREENSHOT

Accessing the vote UI of the VotingApp

SCREENSHOT


If you curious to see what happened under the hood, you can see a new namespace has been created and the application is running inside this one. 

```
```

Within this same namespace we would also get the db-creds secret that is now part of the application.

In this part we have defined the secret directly in the Acornfile but there are cases where we want to create a secret externally, this is what we will explore in the next part.

## Managing secrets from the command line

Using the following command we can get the list of actions we can perform on secrets from the cli:

```
acorn secret --help
```

First we create a secret named postgres-credential, of type basic, which contains 2 keys (username and password) and the associated values:

```
acorn secrets create --type basic --data username=postgres --data password=postgres postgres-credentials
```

We can list the existing secrets:

This command provides an output similar to the one below.

```
```

Once created we can easily retrieve the values:

```
acorn secret expose postgres_credentials
```

As we can see, nothing prevents us from retrieving the values of the secret in plain text once it is created:

```
```

Acorn added an encryption feature in the latest release thus making it possible to encrypt a secret. Let’s then encrypt the postgres-credentials secret:

```
acorn secret encrypt postgres-credentials
```

We will be returned the encrypted version as follow (note the ACORNENC string at the beginning):

```
```

Behind the hood, the encryption is done using the cluster’s public key which can be obtained with the following command:

```
acorn info
```

This provide an output similar to that one, the encryption key being defined under the .namespace.publicKeys.keyID property:

```
```

We can then provide the db credential through this encrypted secret to the db, worker and result containers. This can be done either by passing the secret as an arg to the Acorn image or by providing it to the app at runtime. As we haven’t defined any args in the Acornfile (usage of args will be explained in a future post of the series) we will provide the secret at runtime. For that purpose we use the -s flag and provide the name of the secret we want to use followed by the name of the secret it should be bound to.

We can now run the application and make sure it uses the external postgres-credentials secret with the following command:

```
acorn run -s postgres-credentials:db-creds .
```

As we’ve done previously we could verify the app is working fine and then vote for our favorite pet.

## Summary

In this section we explained how to define and use a secret of type basic in the Acornfile and also how we can use a secrets created from the command line instead. We focused on the connection to the db container but we could use the same approach to secure the connection to redis. In that case we would rather use a secret of type template because redis defines usernames and passwords as acls within a configuration file.

In the next article of the series we will build on this example and use the acorn volumes to add persistent storage to both databases.