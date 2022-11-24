In this step we will use the *router* feature which was recently added in Acorn and which allows to access several containers through a single http endpoint.

In the current version of the Acornfile, when you run the application it generates 2 http endpoints, one for each user facing UI. For instance, in the previous steps you already got endpoints similar to the following ones:

- vote UI: http://resultui-vote-f1825499e037.5its5i.alpha.on-acorn.io
- result UI: http://voteui-vote-c7bc34b6f316.5its5i.alpha.on-acorn.io

You were then able to access the Vote and Result using those endpoints.

Let's now add the following *routers* top level key in the Acornfile

```
routers: 
  voting: {
    routes: {
        "/vote": {
            pathType: "prefix"
            targetServiceName: "voteui"
            targetPort: 80
        }
        "/result": {
            pathType: "prefix"
            targetServiceName: "resultui"
            targetPort: 80
        }
    }
}
```

When the path of a request is either */vote* or */result* the router is in charge of forwarding that request to the *voteui* or *resultui* containers respectively, on port 80.

Note: the definition of the routers can also be defined in a more concise way as follows:

```
routers: 
  voting: {
    routes: {
        "/vote": "voteui:80"
        "/result": "resultui:80"
    }
  }
}
```

You can now run the application using this new version of the Acornfile:

```
acorn run -n vote .
```

After a couple of seconds you will be returned 3 endpoints, they are similar to the following ones:

- http://resultui-vote-f1825499e037.5its5i.alpha.on-acorn.io
- http://voteui-vote-c7bc34b6f316.5its5i.alpha.on-acorn.io
- http://voting-vote-fba3393a0c9a.5its5i.alpha.on-acorn.io

The 2 first ones are similar to the endpoints you get previously (to access the *voteui* or the *resultui* containers directly). The last one was created because of the definition of the router. Using this last endpoint the vote UI can then be accessed via ```http://voting-vote-fba3393a0c9a.5its5i.alpha.on-acorn.io/vote```, the result UI via ```http://voting-vote-fba3393a0c9a.5its5i.alpha.on-acorn.io/result```.

<details>
  <summary markdown="span">If you curious about...</summary>

...what happened under the hood, you would see a new pod was created in the same namespace as the one the application's containers are running in. This pod contains a unique container based on *nginx* whose configuration is retrieved from a ConfigMap. The configuration comes directly from the specification of the routers key. Below is the content of the ConfigMap created in this example:

```
$ kubectl get cm voting-44613d12 -n vote-7dfd932e-d59 -o yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    apply.acorn.io/applied: H4sIAAAAAAAA/3yQza7aMBCFX8WadUIgPyRxxaLLqtsukaqJPQGrztiyTQpCvHuVVFw23Lu0zqdz/M0dNCYEeQfleDQnkBApzBTE/cjWxEQsum23/XZk6xQm41gcRBEoXmxaGCF8cNfbb48xinNKXhZFqOv9rtK7Eh4ZTJTwuYHMLq0lcX16b28bVC7wxrjC/WUK+Wn+AxIMJwqM9pXOu0z8NKwP373/wTEhK4LsfQfjRCBhdulLJHpUC7dGn4HxMuTKcaJrAgmLkQq0OvwyE8WEkwfJF2szsDiQfWd2xngGCWpPuht1M2is2lFV+77ssawH6uq2oe1AzaBaXdXLyEvB8Cn/OOj/4PnxRTBv9aj7qqRcNz08Hv8CAAD//5Lj5EbSAQAA
    apply.acorn.io/owner-gvk: internal.acorn.io/v1, Kind=AppInstance
    apply.acorn.io/owner-name: vote
    apply.acorn.io/owner-namespace: acorn
    apply.acorn.io/owner-sub-context: ""
  creationTimestamp: "2022-11-24T13:36:20Z"
  labels:
    apply.acorn.io/hash: c6ed8fd5bda37fc36929a24be8475e0be5bc7d34
  name: voting-44613d12
  namespace: vote-7dfd932e-d59
  resourceVersion: "7528"
  uid: a7e92668-b58e-43c0-b074-6eb12a300e07
data:
  config: |
    server {
    listen 8080;
    location = /result {
      proxy_pass http://resultui:80;
    }
    location /result/ {
      proxy_pass http://resultui:80;
    }
    location = /vote {
      proxy_pass http://voteui:80;
    }
    location /vote/ {
      proxy_pass http://voteui:80;
    }
    }
```
</details>


[Previous](./development_mode.md)  
