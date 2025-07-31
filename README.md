This README will guide you through setting up a Kubernetes cluster which will be managed by FluxCD.
It will walk you through the absolute basics of setting up a cluster, bootstrapping it, and setting up some core
services. After that, you can start adding functionality of your own.

## Prerequisites

You need the following software installed on your local machine:

- [kubectl](https://kubernetes.io/docs/tasks/tools/) - the Kubernetes command-line tool.
- [flux](https://fluxcd.io/docs/installation/) - a tool to manage Kubernetes clusters declaratively.
- A [GitHub Personal Access Token](https://github.com/settings/tokens) with the `repo` scope to bootstrap FluxCD.

Optional, but recommended:

- [task](https://taskfile.dev/) - a task runner to manage the setup of your cluster.
- [age](https://age-encryption.org/) - a simple, modern and secure file encryption tool.
- An AWS Account to set up reliable storage for backups, external secrets management, and more (optional).

## Setting up your cluster

There are multiple options to set up a Kubernetes cluster. This README will provide you with instructions
for the following:

- [minikube](https://github.com/kubernetes/minikube) – a tool that makes it easy to run Kubernetes locally.
  It runs a single-node Kubernetes cluster inside a VM on your laptop.
- [kind](https://github.com/kubernetes-sigs/kind) – Kubernetes in Docker. All you need is Docker installed.
- Setting up k3s with [k3sup](https://github.com/alexellis/k3sup). The most powerful option that will enable you to
  scale to a multi-node cluster on relatively low-end hardware.

### Installation and Cluster Launch

#### minikube

**Install (macOS with Homebrew):**

```sh
brew install minikube
```

**Install (Linux):**

```sh
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

[Official minikube installation guide](https://minikube.sigs.k8s.io/docs/start/)

**Create the cluster**

```sh
minikube start
```

This will create and start a single-node Kubernetes cluster locally. Verify that the cluster is running with:

```sh
kubectl cluster-info
```

#### kind

**Install (macOS with Homebrew):**

```sh
brew install kind
```

**Install (Linux):**

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

[Official kind installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/)

**Create a cluster:**

```sh
kind create cluster --name kind
```

This will spin up a Kubernetes cluster in Docker containers. Verify that the cluster is running with:

```sh
kubectl cluster-info --context kind-kind
```

#### k3s with k3sup

This guide will cover the installation of k3s on your local machine. For remote setup, refer
to the official k3sup documentation.

**Install k3sup (macOS/Linux):**

```sh
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

[Official k3sup installation guide](https://github.com/alexellis/k3sup#install)

**Create a cluster**

```sh
k3sup install --user $USER --ip $IP
```

This will install k3s and fetch the kubeconfig file. To use `kubectl`, export the kubeconfig:

```sh
kubectl cluster-info
```

You are now ready to use `kubectl` with your chosen cluster.

## Bootstrapping the cluster with FluxCD

Once your cluster is up and running, you can bootstrap it with FluxCD to enable GitOps-based management.

If the repository `kubernetes-workshop` does _not_ exist, FluxCD will create it for you during the bootstrap process.

Afterward, you can bootstrap FluxCD with the following command:

```sh
export $GITHUB_TOKEN=<your-github-token>
export $GITHUB_USER=<your-github-username>

task bootstrap-flux

# equivalent to:
# flux bootstrap github \
#           --owner=$GITHUB_USER \
#           --repository=kubernetes-workshop \
#           --branch=main \
#           --path=clusters/demo \
#           --personal
```

This command will install FluxCD controllers and configure them to sync with your repository.
For other Git providers or advanced options, see the
[FluxCD bootstrap documentation](https://fluxcd.io/docs/cmd/flux_bootstrap/).

After bootstrapping, Flux will continuously reconcile your cluster state with your Git repository.
