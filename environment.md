In this step we will set up the environment that will be used for the workshop.

## Kubernetes cluster

To illustrate this article, we will use a one node k3s cluster.
k3s comes with an Ingress Controller and a StorageClass by default which is great as both are needed by Acorn so it can expose and provide storage to the application containers.

### Provisionning a VM

[Multipass](https://multipass.run) is a great tool (available on MacOS, Windows and Linux) to spin up Ubuntu VM in a breeze. We will use Multipass in this workshop but you can use the provisonning tool of your choice to create a Ubuntu VM.

The following command launch a VM named k3s with a couple of additional options:

```
multipass launch -n k3s -c 2 -d 10G -m 2G
```

### Install k3s

First run a shell in this new VM, this can be done easily using the following command with multipass:

```
multipass shell k3s
```

Next install k3s:

```
curl -sSL https://get.k3s.io | sh
```

Next configure kubectl to it uses the kubeconfig file created by k3s:

```
mkdir -p $HOME/.kube
sudo mv -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then you should be able to access your *one node* cluster:

```
$ kubectl get node
NAME   STATUS   ROLES                  AGE   VERSION
k3s    Ready    control-plane,master   25s   v1.25.3+k3s1
```

## Acorn

Now you have a local Kubernetes, you will install Acorn inside of it.

First we download and install the Acorn CLI following the installation documentation, on MacOS and Linux this handy installation script can be used:

```
curl https://get.acorn.io | sh
```

You should get an output similar to the following one (your Acorn version might be different though):

```
[INFO]  Finding release for channel latest
[INFO]  Using v0.3.1 as release
[INFO]  Downloading hash https://github.com/acorn-io/acorn/releases/download/v0.3.1/checksums.txt
[INFO]  Downloading archive https://github.com/acorn-io/acorn/releases/download/v0.3.1/acorn-v0.3.1-linux-arm64.tar.gz
[INFO]  Verifying binary download
[INFO]  Installing acorn to /usr/local/bin/acorn
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

Next install the Acorn server side components in the cluster:

```
acorn install
```

This should return a content similar to the following one:

```
  ✔  Running Pre-install Checks
  ✔  Installing ClusterRoles
  ✔  Installing APIServer and Controller (image ghcr.io/acorn-io/acorn:v0.3.1)
  ✔  Waiting for controller deployment to be available
  ✔  Waiting for API server deployment to be available
  ✔  Running Post-install Checks
  ✔  Running Post-install Checks
  ✔  Installation done
```

Acorn is installed and ready to manage containerized applications. I

[Previous](./acorn.md)
[Next](./votingapp.md)