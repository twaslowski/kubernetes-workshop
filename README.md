This README will guide you through setting up a Kubernetes cluster which will be managed by FluxCD.
It will walk you through the absolute basics of setting up a cluster, bootstrapping it, and setting up some core
services. After that, you can start adding functionality of your own.

## Prerequisites

You **need** to perform the following steps before you can follow this guide:

- Install [kubectl](https://kubernetes.io/docs/tasks/tools/) - the Kubernetes command-line tool.
- Install [flux](https://fluxcd.io/docs/installation/) - a tool to manage Kubernetes clusters declaratively.
- Create a [GitHub Personal Access Token](https://github.com/settings/tokens) with the `repo` scope to bootstrap FluxCD.
- Fork this repository to your own GitHub account. This will be used to store the configuration of your cluster.

You also **should** have a rudimentary understanding of Kubernetes concepts such as
Pods, Deployments, Services, and Namespaces. If you don't, [Section 0](#step-0-understanding-the-problem-statement)
will cover some of the basics, but you _really_ should check out the Kubernetes documentation or a beginner's guide
first. 

The official tutorial is a great place to start: [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics).

You **should**, but do not have to, perform the following steps:

- Install [task](https://taskfile.dev/) - this guide provides some convenience commands using Task.
- Install [age](https://age-encryption.org/) - for sealed secrets management.
  You can use GPG, but this guide will not cover it.
- An AWS Account to set up reliable storage for backups, external secrets management.
  You can also use different cloud providers or a minio instance.

# Setting up your cluster

There are multiple options to set up a Kubernetes cluster. This README will provide you with instructions
for the following:

- [minikube](https://github.com/kubernetes/minikube) – a tool that makes it easy to run Kubernetes locally.
  It runs a single-node Kubernetes cluster inside a VM on your laptop.
- [kind](https://github.com/kubernetes-sigs/kind) – Kubernetes in Docker. All you need is Docker installed.
- Setting up k3s with [k3sup](https://github.com/alexellis/k3sup). The most powerful option that will enable you to
  scale to a multi-node cluster on relatively low-end hardware.

## Installation and Cluster Launch

### minikube

**1. Install (macOS with Homebrew):**

```sh
brew install minikube
```

**Install (Linux):**

```sh
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**2. Create the cluster**

```sh
minikube start
```

This will create and start a single-node Kubernetes cluster locally. Verify that the cluster is running with:

```sh
kubectl cluster-info
```

### kind

**1. Install (macOS with Homebrew):**

```sh
brew install kind
```

**Install (Linux):**

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**2. Create a cluster:**

```sh
kind create cluster --name kind
```

This will spin up a Kubernetes cluster in Docker containers. Verify that the cluster is running with:

```sh
kubectl cluster-info --context kind-kind
```

### k3s with k3sup

This guide will cover the installation of k3s on your local machine. For remote setup, refer
to the official k3sup documentation.

**1. Install k3sup (macOS/Linux):**

```sh
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

**2. Create a cluster**

```sh
k3sup install --user $USER --ip $IP
```

This will install k3s and fetch the kubeconfig file. To use `kubectl`, export the kubeconfig:

```sh
kubectl cluster-info
```

You are now ready to use `kubectl` with your chosen cluster.

# Step 0: Understanding the Problem Statement

If you are a complete beginner, you might be wondering why we need a tool like FluxCD to manage our Kubernetes cluster.
This section covers that. If you already have experience with Kubernetes, you can skip this section.

## Let's create a Pod

Let's start with a simple example. We will create a Pod that runs a basic web server.

Create a file named `nginx.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

Now you can run `kubectl apply -f nginx.yaml` to create the Pod. You can verify that the Pod is running with:

```sh
kubectl get pods
```

Let's image you now need to _modify_ your Deployment. You want to change the image to a different version,
or you want to add an environment variable. Let's say you want to update the image to `nginx:latest`.

The most straightforward way to do this is to edit the `nginx.yaml` file and then again run:

```sh
kubectl apply -f deployment.yaml
```

Of course, manually updating your Cluster every time you want to change something is not very efficient.
This is where GitOps comes into play. Instead of manually applying changes to your cluster, you can store your
Kubernetes manifests in a Git repository and use a tool like FluxCD to automatically apply those changes to your cluster.

Let's set up FluxCD to manage our cluster.

## Bootstrapping the cluster with FluxCD

Once your cluster is up and running, you can bootstrap it with FluxCD to enable GitOps-based management.

Afterward, you can bootstrap FluxCD with the following command:

```sh
export GITHUB_TOKEN=<your-github-token>
export GITHUB_USER=<your-github-username>

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

You can verify that the installation was successful by checking the status of the Flux controllers:

```sh
flux get all
# or
kubectl get pods -n flux-system
```

Congratulations! Flux will now continuously reconcile your cluster state with your Git repository. Let's add some stuff!

## Creating and updating an application

In this section, we will create a simple application and deploy it to our cluster using FluxCD.
In [Section 0](#step-0-understanding-the-problem-statement), we created a Deployment manually.
Now, let's manage it with FluxCD.
Delete your original deployment, so we can start fresh. Then move your `nginx.yaml` file to the `clusters/demo`
directory in your forked repository.

```sh
kubectl delete deployment nginx-deployment
mv nginx.yaml clusters/demo/nginx.yaml

# Commit and push the changes
# Recommended remote structure: `upstream` for the original repository, `origin` for your fork
git add clusters/demo/nginx.yaml
git commit -m "Add nginx deployment"
git push origin main
```

FluxCD will automatically detect the new file and apply it to your cluster.