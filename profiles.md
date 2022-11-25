Sometimes we need to have the application configured differently, for instance between different environments. Args and profiles are 2 Acorn concepts which are often used together to provide a dynamic configuration to an application so it can defined a different set of default for each environment.

- *args* defines arguments that can be modified at build or runtime by the user
- *profiles* 

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

Next modify the definition of the *voteui* container so it looks as follows, the number of *voteui* containers now depends on the value of the *replicas* arguments:

```
voteui: {
    if args.dev {
      dirs: "/usr/share/nginx/html": "./vote-ui"
    } 
    ports: publish : "80/http"
    scale: args.replicas
  }
```

Test the following actions:

- run the app without args nor profile information

make sure the following command uses the default value of the *replicas* args, it should then create 3 *voteui* containers:

```
acorn run -n vote --update .
```

Note: you can verify the running containers with ```acorn all``` (which lists all the Acorn resources) or with ```acorn containers``` (which only lists containers)

- running the app overwriting the default args

The following command will not use the default value of the *replicas* args but it will use the user supplied value instead. Make sure it create 5 *voteui* containers:

```
acorn run -n vote --update . --replicas=5
```

- running the app specifying a profile

The following command will not use the default value of the *replicas* args but it will use the value defined in the *test* profile instead. Make sure it creates 2 containers for the *vote-ui* microservice:

```
acorn run -n vote --update --profile test .
```

We only used the *voteui* container to illustrate the usage of args / profile but we could have used other stateless containers in the same way.

Note: you can find more information about Arguments and Profiles in [the official documentation](https://docs.acorn.io/authoring/args-and-profiles)

[Previous](./router.md)  
