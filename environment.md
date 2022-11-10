In the next steps we will set up the environment we will use for the workshop.

## Kubernetes cluster

To illustrate this article, we will use a one node k3s cluster with the Traefik Ingress Controller installed. An Ingress Controller is needed for Acorn to expose applications, also a StorageClass is needed for Acorn to provide storage to containers that need it.

### Provisionning a VM

```
multipass launch -n k3s
```

Run a shell in the new VM:

### Install k3s

```
multipass shell k3s
```

Install k3s inside that one:

```
curl -sSL https://get.k3s.io | sh
```

### Get the kubeconfig

```
```


## Acorn

First we install the Acorn CLI following the installation documentation. On MacOS and Linux this handy installation script can be used:

```
$ curl https://get.acorn.io | sh
```

Running the acorn command without any parameters returns the full list of commands available to manage Acorn’s applications. We will use a couple of those commands in the next steps.

```
$ acorn
Acorn: Containerized Application Packaging Framework

Usage:
  acorn [flags]
  acorn [command]

Available Commands:
  all         List (almost) all objects
  app         List or get apps
  build       Build an app from a Acornfile file
  check       Check if the cluster is ready for Acorn
  container   List or get running containers
  credential  Manage registry credentials
  exec        Run a command in a container
  help        Help about any command
  image       List images
  info        Info about acorn installation
  install     Install and configure acorn in the cluster
  login       Add registry credentials
  logout      Remove registry credentials
  logs        Log all pods from app
  pull        Pull an image from a remote registry
  push        Push an image to a remote registry
  render      Evaluate and display an Acornfile with args
  rm          Delete an app, container, or volume
  run         Run an app from an image or Acornfile
  secret      Manage secrets
  start       Start an app
  stop        Stop an app
  tag         Tag an image
  uninstall   Uninstall acorn and associated resources
  update      Update a deployed app
  volume      List or get volumes
  wait        Wait an app to be ready then exit with status code 0

Flags:
  -A, --all-namespaces      Namespace to work in
      --context string      Context to use in the kubeconfig file
  -h, --help                help for acorn
      --kubeconfig string   Location of a kubeconfig file
      --namespace string    Namespace to work in (default "acorn")
  -v, --version             version for acorn

Use "acorn [command] --help" for more information about a command.
```

Next we install the Acorn server side components in the cluster:

```
$ acorn install
This should give a result like the following one:

  ✔  Running Pre-install Checks
  ✔  Installing ClusterRoles
  ✔  Installing APIServer and Controller (image ghcr.io/acorn-io/acorn:v0.2.1)
  ✔  Waiting for controller deployment to be available
  ✔  Waiting for API server deployment to be available
  ✔  Installation done
````

Acorn is installed and ready to manage containerized applications !

