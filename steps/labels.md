It is often usefull to add labels on application's resources. Labels allow to group resources in a logical way and also to filter them. As you will see below, Acorn makes it very easy to add labels on the whole application or on indivual pieces.

First modify the Acornfile adding the *labels* top-level key at the top of the file:

```
labels: {
    application: "votingapp"
}
```

Next modify the definition of each container adding a new label property with *component* as the key and the container's name as the associated value. The following shows the changes done on the *voteui* container:

```
  voteui: {
    labels: {
      component: "voteui"
    }
    if args.dev {
      dirs: "/usr/share/nginx/html": "./vote-ui"
    }
    build: {
      context: "./vote-ui"
    }
    ports: publish : "80/http"
    scale: args.replicas
  }
```

Once you've added the new label for each container, update the application:

```
acorn run -n vote --update  .
```

Under the hood this will add the specified labels on the application's resources.

<details>
  <summary markdown="span">If you curious about...</summary>

...what happened under the hood, you can see that the Pods created now have 2 additional labels:

- application
- component

Those labels were added on top of the labels automatically set when running the acorn application:

- acorn.io/app-name
- acorn.io/app-namespace
- acorn.io/container-name
- acorn.io/managed
- port-number.acorn.io/xxx
- service-name.acorn.io/yyy

```
$ kubectl get po --show-labels
NAME                        READY   STATUS    RESTARTS   AGE     LABELS
resultui-58c45b65cc-h276s   1/1     Running   0          3m48s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=resultui,acorn.io/managed=true,application=votingapp,component=resultui,pod-template-hash=58c45b65cc,port-number.acorn.io/80=true,service-name.acorn.io/resultui=true
voteui-7d754fff94-bbmjj     1/1     Running   0          3m47s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=voteui,acorn.io/managed=true,application=votingapp,component=voteui,pod-template-hash=7d754fff94,port-number.acorn.io/80=true,service-name.acorn.io/voteui=true
vote-cc685d54f-g8xmt        1/1     Running   0          3m48s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=vote,acorn.io/managed=true,application=votingapp,component=vote,pod-template-hash=cc685d54f,port-number.acorn.io/5000=true,service-name.acorn.io/vote=true
voting-679bbf8f6-7f86f      1/1     Running   0          3m48s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/managed=true,acorn.io/router-name=voting,application=votingapp,pod-template-hash=679bbf8f6,port-number.acorn.io/8080=true,service-name.acorn.io/voting=true
voteui-7d754fff94-92sn8     1/1     Running   0          3m40s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=voteui,acorn.io/managed=true,application=votingapp,component=voteui,pod-template-hash=7d754fff94,port-number.acorn.io/80=true,service-name.acorn.io/voteui=true
result-5b545474fc-qj45k     1/1     Running   0          3m43s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=result,acorn.io/managed=true,application=votingapp,component=result,pod-template-hash=5b545474fc,port-number.acorn.io/5000=true,service-name.acorn.io/result=true
worker-8f5bf57d7-ctcf4      1/1     Running   0          3m43s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=worker,acorn.io/managed=true,application=votingapp,component=worker,pod-template-hash=8f5bf57d7
voteui-7d754fff94-vbtqx     1/1     Running   0          3m35s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=voteui,acorn.io/managed=true,application=votingapp,component=voteui,pod-template-hash=7d754fff94,port-number.acorn.io/80=true,service-name.acorn.io/voteui=true
redis-cc8885755-w26d4       1/1     Running   0          3m44s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=redis,acorn.io/managed=true,application=votingapp,component=redis,pod-template-hash=cc8885755,port-number.acorn.io/6379=true,service-name.acorn.io/redis=true
db-6bcd948cb-jz4jl          1/1     Running   0          3m35s   acorn.io/app-name=vote,acorn.io/app-namespace=acorn,acorn.io/container-name=db,acorn.io/managed=true,application=votingapp,component=db,pod-template-hash=6bcd948cb,port-number.acorn.io/5432=true,service-name.acorn.io/db=true
```
</details>

Note: you can find more information about labels in [the official documentation](https://docs.acorn.io/authoring/labels)

[Previous](./profiles.md)  
[Next](./acorn_image.md)