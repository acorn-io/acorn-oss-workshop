In the previous part you built your own image of the application and sent it to the DockerHub, we will now see how Acorn can be used to automatically upgrade the running application when a new version of the image is available.

First run the application with a SemVer pattern as follows:

Note: replace the placeholder *YOUR_DOCKERHUB_USERNAME* with your actual DockerHub's username

```
acorn run -n vote docker.io/YOUR_DOCKERHUB_USERNAME/acorn-workshop:v#.#.#
```

As usual, you should receive http endpoints to access both *voteui* and *resultui* container. Run the following command to verify the application is using version *v1.0.0* of your image:

```
$ acorn app
NAME      IMAGE                                        HEALTHY   UP-TO-DATE   CREATE    ENDPOINTS  MESSAGE
vote      index.docker.io/lucj/acorn-workshop:v1.0.0   9         9            65s ago   ...        OK
```

Next modify the application a bit. For instance you can change the color of the *voteui* background modifying the following part in *vote-ui/content/content.css*:

```
...
html,body{
  margin: 0;
  padding: 0;
  background-color: #F7F8F9;      <- this needs to be modified
  height: 100vh;
  font-family: 'Open Sans';
}
```

so it looks as follows:

```
html,body{
  margin: 0;
  padding: 0;
  background-color: red;
  height: 100vh;
  font-family: 'Open Sans';
}
```

Next build the new version of the application specifying *v1.0.1* in the image's tag:

```
acorn build -t docker.io/YOUR_DOCKERHUB_USERNAME/acorn-workshop:v1.0.1 .
```

Then push this new version to the DockerHub:

```
acorn push docker.io/YOUR_DOCKERHUB_USERNAME/acorn-workshop:v1.0.1
```

For a few tens of seconds the application is still running in version v1.0.0

```
$ acorn app
NAME      IMAGE                                       HEALTHY   UP-TO-DATE   CREATED   ENDPOINTS  MESSAGE
vote      index.docker.io/lucj/vote-workshop:v1.0.0   7         7            14m ago   ...        OK
```

Then it is automatically upgraded from *v1.0.0* to *v1.0.1*

```
$ acorn app
NAME      IMAGE                                       HEALTHY   UP-TO-DATE   CREATED   ENDPOINTS MESSAGE
vote      index.docker.io/lucj/vote-workshop:v1.0.1   7         7            15m ago   ...       OK
```

Using the http endpoint of the *voteui* container you should now see the new color of the background

![Vote UI](./images/upgrade/vote-ui.png)

Each time a newer SemVer tag is pushed to the registry the application will be upgraded with the new image.

Change back the content of *vote-ui/content/content.css* so the application looks a little bit nicer for the following steps :)

Once you'r done, remove the application

```
acorn rm vote -af
```

Note: you can find more information about Automatic Ugrades in [the Acorn documentation](https://docs.acorn.io/running/auto-upgrades)

[Previous](./acorn_image.md)  
[Next](./domain.md)