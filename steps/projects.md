A project allows to group resources like applications, images, volumes, and secrets together. A project can be seen as a context under which we can interact with Acorn applications and related resources.  

Until now you have run the app in the default project named *acorn*, this is the default project as shows the following command:

```
$ acorn project
NAME      DEFAULT   REGIONS
acorn     *         local*
```

First create a new project named *voting*:

```
acorn project create voting
```

Verify it appears in the list of projects:

```
$ acorn project
NAME      DEFAULT   REGIONS
acorn     *         local*
voting              local*
```

Set *voting* as the default project:

```
acorn project use voting
```

This *voting* project should now appear as the default one:

```
$ acorn project
NAME      DEFAULT   REGIONS
acorn               local*
voting    *         local*
```

Run the application inside of this project

```
acorn run -n vote .
```

You will be returned http endpoints to access both *voteui* and *resultui* web frontends as before.
Once you have vote for your favorite pet once again you can delete the project, this will also delete all related resources.

```
acorn project rm voting
```

You can then come back to the default project:

```
acorn project use acorn
```

Note: you can find more information about projects in [the Acorn documentation](https://docs.acorn.io/running/projects)

[Previous](./job.md)  
[Next](./acorn_image.md)