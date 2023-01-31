In the previous step you create a first version of the Acornfile for the VotingApp and run the whole application. In the current step you will use several commands to interact with the app, commands which can also help to troubleshoot the app.

## Running the application

Let's run the application once again:

```
acorn run -n vote .
```

If should only take a few tens of seconds for the application to be up and running.

## Inspecting the app

Once the application is running we can have app related information, like the endpoints exposing the app, with the following command:

```
$ acorn app
NAME      IMAGE          HEALTHY   UP-TO-DATE   CREATED   ENDPOINTS                                                                                                                                  MESSAGE
vote      17d83c8edf12   7         7            73s ago   http://resultui-vote-f1825499.gek0vg.alpha.on-acorn.io => resultui:80, http://voteui-vote-c7bc34b6.gek0vg.alpha.on-acorn.io => voteui:80   OK
```

Using the following command we can also list the running containers:

```
$ acorn containers
NAME                           APP       IMAGE                                                                     STATE     RESTARTCOUNT   CREATED    MESSAGE
vote.voteui-8689cb8f88-4n2h7   vote      sha256:89cb11828c2a09866d20bab7c0e73f2a97f233c0275ed731b2b2fab5198916b8   running   0              2m ago
vote.db-7d668d895f-945cs       vote      postgres:13.2-alpine                                                      running   0              2m1s ago
vote.redis-866df78d85-tlbbl    vote      redis:6.2-alpine3.13                                                      running   0              2m1s ago
vote.result-7c94c65dfc-cwx72   vote      sha256:75f550973621d56e25655aa20862d797d3984b286fb1f47e627892e54b3f6741   running   0              2m1s ago
vote.resultui-6db89bfc-htklw   vote      sha256:bce5a745596864b40b4da3eccd09ee677a6faa90c485d77a61be23aa6269c53a   running   0              2m1s ago
vote.vote-76b6dd8b79-fjscp     vote      sha256:86cda2b9d85b277c22ad4b93fa05cf05f3485df450e738c10373c62cfd95a4e1   running   0              2m1s ago
vote.worker-6c56f7f587-nwg7r   vote      sha256:6d08570ee5072485d7de6ed6b9ceea4dc9177a627819f214bd5b58808c4f2799   running   0              2m1s ago
```

## Getting the logs

Acorn allows us to get the logs of the whole application:

```
acorn logs vote
```

![Application logs](./images/ops/app-logs.png)


It also allows to get the logs of a single container as illustrated with the following command which get the logs of the *voteui* container (which fullname is *vote.voteui-8689cb8f88-4n2h7* as shown in the list of containers above):

```
acorn logs vote.voteui-8689cb8f88-4n2h7
```

![Container logs](./images/ops/container-logs.png)

## Running command in a container

From the command line we can launch a command in a running container. The command below runs a *sh* shell in the *voteui* container and list the processes runnning inside that one:

```
acorn exec vote.voteui-8689cb8f88-4n2h7
```

![Container exec](./images/ops/container-exec.png)

In this step we focused on the application itself and saw how to get information and interact with it. In the next step we will enhance the Acornfile using Acorn secrets.

[Previous](./acornfile.md)  
[Next](./secret.md)