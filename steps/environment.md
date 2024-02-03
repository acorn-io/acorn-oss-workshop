In this step we will set up the environment that will be used for the workshop.

Note: in this lab you are asked to setup the environment yourself, but in the context of your organization you may already have a Kubernetes cluster (with Acorn installed inside of it) running and managed by an ops/sre team. We've tried to make the instructions below as simple as possible to quickly eliminate the installation step :)

## Kubernetes cluster

To illustrate this article, we will use a one node k3s cluster.
k3s comes with an Ingress Controller and a StorageClass by default which is great as both are needed by Acorn so it can expose and provide storage to the application containers.

## Quick path

If you are on Linux / MacOS, first install [Multipass](https://multipass.run) a tool which allows to spin up local Ubuntu VM, then run the following commands. In a couple of minutes you will have access to a one-node k3s with Acorn installed inside of it.

```
curl -sSLO https://luc.run/acorn.sh
chmod +x ./acorn.sh
./acorn.sh 
```

Note: the same script will have its ps1 counterpart soon so you will be able to use this quick path on Windows too

Once the installation is done, you can run a shell in the newly created VM:

```
multipass shell acorn
```

Within this shell, make sure Acorn installation was done correctly:

```
ubuntu@acorn:~$ acorn check
NAME                  PASSED    MESSAGE
RBAC                  true      User can create namespaces
NodesReady            true      All nodes are ready
DefaultStorageClass   true      Found default storage class local-path
IngressCapability     true      Ingress is ready
Exec                  true      Successfully executed command in container replica
  ✔  Checks PASSED
```

If everything went fine you can skip the following and go directly to the [presentation of the Voting Application](./votingapp.md), otherwise feel free to read the rest of this chapter if you want to have a better understanding of the installation process.

## Detailed path

If you don't want (or just can't) to go using the quick path, you can follow the steps below to setup your own environment.

### Provisionning a VM

First, you need to have a virtual machine, you can either spin up a local VM or create one on the infrastructure of a cloud provider.

#### Local

In order to create a local VM, you can use [Multipass](https://multipass.run), it's a great tool (available on MacOS, Windows and Linux) to spin up Ubuntu VM in a breeze. We will use Multipass in this workshop but you can use the provisonning tool of your choice to create a Ubuntu VM.

The following command launch a VM named k3s with a couple of additional options:

```
multipass launch -n k3s -c 4 -d 20G -m 4G
```

#### Cloud provider

In case you have access to a cloud provider (AWS, Google GCP, DigitalOcean, Exoscale, Scaleway, ...), you can spin up a VM on their infrastructre.

Using your favorite cloud provider, run a small VM something like 2G RAM, 2 vcpus. This will only cost you a couple of dollars for the entire workshop. Do not forget to delete the VM after the workshop though !

### Run a shell in the VM

Next, you need to run a shell in the newly proviosnned VM.

- if you use Multipass, you can run a shell in the VM using the following command:

```
multipass shell k3s
```

- if you use a VM provisionned on the cloud provider you'll need to use a ssh client with the credentials or keys provided by the cloud provider.

### Install k3s

Next, install k3s with the following command:

```
curl -sSL https://get.k3s.io | sh
```

Next configure kubectl to it uses the kubeconfig file created by k3s:

```
mkdir -p $HOME/.kube
sudo mv -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then you will be able to access your *one node* cluster:

```
kubectl get node
```

You should get a return similar to the one below:

```
NAME   STATUS   ROLES                  AGE   VERSION
k3s    Ready    control-plane,master   16s   v1.28.5+k3s1
```

Note: your version of k3s could be slightly different

## Acorn

Now you have a local Kubernetes, you will install Acorn inside of it.

First we download and install the Acorn CLI following the installation documentation, on MacOS and Linux this handy installation script can be used:

```
curl https://get.acorn.io | sh
```

You should get an output similar to the following one (your Acorn version might be different though):

```
[INFO]  Finding release for channel latest
[INFO]  Using v0.10.0 as release
[INFO]  Downloading hash https://github.com/acorn-io/acorn/releases/download/v0.10.0/checksums.txt
[INFO]  Downloading archive https://github.com/acorn-io/acorn/releases/download/v0.10.0/acorn-v0.10.0-linux-arm64.tar.gz
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
  all          List (almost) all objects
  build        Build an app from a Acornfile file
  check        Check if the cluster is ready for Acorn
  container    Manage containers
  copy         Copy Acorn images between registries
  credential   Manage registry credentials
  dashboard    Open the web dashboard for the project
  dev          Run an app from an image or Acornfile in dev mode or attach a dev session to a currently running app
  edit         Edits an acorn or secret interactively. The things you can change with acorn edit are the same things you can set via the CLI when running acorn run.
  events       List events about Acorn resources
  exec         Run a command in a container
  fmt          Format an Acornfile
  help         Help about any command
  image        Manage images
  info         Info about acorn installation
  install      Install and configure acorn in the cluster
  job          Manage jobs
  login        Add registry credentials
  logout       Remove registry credentials
  logs         Log all workloads from an app
  offerings    Show infrastructure offerings
  port-forward Forward a container port locally
  project      Manage projects
  ps           List or get apps
  pull         Pull an image from a remote registry
  push         Push an image to a remote registry
  render       Evaluate and display an Acornfile with args
  rm           Delete an acorn, optionally with it's associated secrets and volumes
  run          Run an app from an image or Acornfile
  secret       Manage secrets
  start        Start an app
  stop         Stop an app
  tag          Tag an image
  uninstall    Uninstall acorn and associated resources
  update       Update a deployed Acorn
  version      Version information for acorn
  volume       Manage volumes
  wait         Wait an app to be ready then exit with status code 0

Flags:
      --config-file string   Path of the acorn config file to use
      --debug                Enable debug logging
      --debug-level int      Debug log level (valid 0-9) (default 7)
  -h, --help                 help for acorn
      --kubeconfig string    Explicitly use kubeconfig file, overriding the default context
  -j, --project string       Project to work in

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
  ✔  Installing APIServer and Controller (image ghcr.io/acorn-io/runtime:v0.10.0)
  ✔  Waiting for controller deployment to be available
  ✔  Waiting for API server deployment to be available
  ✔  Waiting for registry server deployment to be available
  ✔  Running Post-install Checks
  ✔  Installation done
```

Acorn is installed and ready to manage containerized applications.

[Previous](./acorn.md)  
[Next](./votingapp.md)