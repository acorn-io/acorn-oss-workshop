In the previous step we enhanced the Acornfile of the VotingApp defining volumes for both db and redis containers. In this step we will use Acorn's development mode which eases the development workflow.

## About development mode

In development mode, Acorn allows to make changes to the source code and see it updated inside the app containers in real time. In this mode Acorn will watch the local directory for changes and synchronize them to the running Acorn app. To activate the development mode we need to use the *-i* flag when running the app as you will do in a bit.

In this step we will focus on the result microservice which is developed with NodeJS. If in development mode, we can mount the code folder in the /app in the container and specify the NODE_ENV environment variable as follows.

```
result: {
    build: "./result"
    ports: "5000/http"
    if args.dev { 
      dirs: "/app": "./result"
      env: "NODE_ENV": "development" 
    }
  }
```

You can now stop the application (if it was already running) and run it in development mode:

```
acorn run -n vote -i .
```

Once the application is up and running you can change an instruction in *server.js* in the *result* folder and see that changes automatically been taken into account.

[Previous](./acorn_image.md)