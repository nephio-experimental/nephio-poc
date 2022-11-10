# Nephio Proof-of-Concept

The Nephio project is building a Kubernetes-based automation platform for
deploying and managing highly distributed, interconnected workloads such as 5G
Network Functions, and the underlying infrastructure on which those workloads
depend.

This repository provides a guide to setting up a Proof-of-Concept version of
Nephio, along with a demo scenario.

## Community

Please see the following resources for more information:
  * Website: [nephio.org](https://nephio.org)
  * Wiki: [wiki.nephio.org](https://wiki.nephio.org)
  * Slack: [nephio.slack.com](https://nephio.slack.com)
  * Governance:
    [github.com/nephio-project/governance](https://github.com/nephio-project/governance)

## Installation Overview

Nephio is very early in its development; there is no release yet. However if you
wish to experiment with the project or contribute to it, the following
instructions will help you get a pre-release version up.

To install and run Nephio, you will need:
  * A Kubernetes cluster.
  * The Kubernetes CLI client, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl).
  * The Kpt CLI client, [kpt](https://kpt.dev/installation/kpt-cli).
  * A Git repository provider. As of now, GitHub and Google Cloud Source
    Repositories are supported.
  * An OAuth 2.0 client ID, if you wish to install the GUI. The GUI only works
    with GKE right now, due to how authentication is done.

### Installation Steps
  1. Install the prerequisite tools on your workstation.
     * [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
     * [kpt](https://kpt.dev/installation/kpt-cli)
  1. Bring Your Own Kubernetes Cluster or [Create a GKE Cluster](#creating-a-gke-cluster)
     * If you bring your own cluster, make sure your `kubectl` context is
       pointing at that cluster as you run the `kpt` and `kubectl` commands that
       follow.
  1. [Install the Nephio Server Components](#installing-the-server-components)
  1. [Install the Nephio Web UI](#installing-the-web-ui) (Optional)
  1. [Create Repositories](#creating-repositories)
  1. [Register Repositories](#registering-repositories)

After that, Nephio will be ready for use.

### Testing Steps
  1. [Create a Workload Cluster](#creating-a-workload-cluster)
  1. [Install Config Sync in Workload Clusters](#installing-config-sync-in-workload-clusters)
  1. [Deploy a Package Workload](#deploying-a-package-workload)

## Creating a GKE Cluster

These instructions are for GKE Autopilot. You can use any Kubernetes cluster,
though. If you are using a different cluster you can skip to the next section.

To use GKE, you will need a Google Cloud account and project, and you will need
to [install gcloud](https://cloud.google.com/sdk/docs/install) on your
workstation.

Once `gcloud` is installed and your GCP project is created, you need to point
`gcloud` at that project:

```
gcloud config set project YOUR_GCP_PROJECT
```

Next, enable the GKE service on the project:
```
gcloud services enable container.googleapis.com
```

Finally, create the cluster, and then configure `kubectl` to point to the
cluster (you can use a different region, if you prefer):

```
# Create the cluster
gcloud container clusters create-auto --region us-central1 nephio
# This will take a few minutes
# Once it returns, configure kubectl with the credentials for the cluster
gcloud container clusters get-credentials --region us-central1 nephio

# See the nodes in your new cluster
kubectl get nodes
NAME                                            STATUS   ROLES    AGE    VERSION
gk3-nephio-default-pool-0e8b5852-c1w5   Ready    <none>   1m5s   v1.22.10-gke.600
gk3-nephio-default-pool-792b3c85-p0f4   Ready    <none>   1m5s   v1.22.10-gke.600
```

## Installing the Server Components

The Nephio software runs within the Kubernetes cluster. First, let's create a
working directory for our package files:

```
mkdir nephio-install
cd nephio-install
```

Next fetch the package using `kpt`, and run any `kpt` functions:

```
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-packages.git/nephio-system
kpt fn render nephio-system
```

Now, we apply the package. If we are using GKE Autopilot, we need to give
some extra time for the deployment, as it may need to spin up new nodes which
takes a while. Thus, we add the `--reconcile-timeout=15m` flag.

```
kpt live init nephio-system
kpt live apply nephio-system --reconcile-timeout=15m --output=table
```

## Installing the Web UI

Currently, we can just run the prototype Config-as-Data UI from the [kpt](https://github.com/GoogleContainerTools/kpt)
project. In time we will build our own UI.

### Installing the Web UI Package

The prototype UI is a separate package, so let's install that now.

```
kubectl create ns nephio-webui
```

Next, we fetch the package, and then execute `kpt fn render` to execute the
`kpt` function pipeline and prepare the package for deployment.

```
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-packages.git/nephio-webui
kpt fn render nephio-webui
```

Then we apply it:

```
kpt live init nephio-webui
kpt live apply nephio-webui --reconcile-timeout=15m --output=table
```

### Accessing the Web UI

For this prototyping, we are not exposing the Web UI via a load balancer
service. This means that the Web UI is only available on an in-cluster IP
address. Thus, we need to port forward via `kubectl` to access the Web UI from
our workstation browser.

```
kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
```

You can now access the Web UI on your workstation by visiting
[http://localhost:7007/config-as-data](http://localhost:7007/config-as-data) in
your browser.

You will be given a choice of OAuth 2.0 providers - Google will be the only
option at this time. Clicking on that will allow you to login using your Google
account, which will then be used as the identity that the Web UI uses to
interact with the Kubernetes server.

## Creating Repositories

Nephio can work with repositories in GitHub or in the Google Cloud Source
Repository service. This example will use GitHub.

There are two types of repositories in Nephio: "blueprint" repositories and
"deployment" repositories. The difference is in the validations performed, and
the intended consumption model of the packages (blueprints) in each type of
repository.

  * Blueprint repositories contain packages that could not be (or at least are
    not intended to be) directly instantiated on a Kubernetes cluster. These
    packages require additional customization in order to become actual, running
    workloads on a cluster.
  * Deployment repositories contain packages that are fully prepared for
    consumption by the API server; also known as "fully hydrated". These are the
    repositories that will be watched by the GitOps deployment tool (e.g.,
    ConfigSync) running in the workload cluster.

The prototype UI adds one additional distinction between "Catalog Blueprint"
clusters and "Blueprint" clusters, with the former being intended for public or
vendor upstream packages, and the latter for local, private organizational
versions of those upstream packages and other organization-local packages.

In Nephio, workload clusters are typically associated with deployment
repositories in a one-to-one fashion. It's not strictly necessary but is the
expected, most common usage model.

To create a GitHub repository, see [the GitHub
Help](https://docs.github.com/en/get-started/quickstart/create-a-repo). Nephio
supports public or private repositories in GitHub. Nephio will need a `main`
branch, so go ahead and have GitHub create the `README.md` for you, which will
create that branch.

You will need to create a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
with `repo` scope to use Nephio. You may want to consider creating a separate
"Nephio Test" user account for this purpose. Use of more selectively scoped
authentication such as per-repo [Deploy Keys](https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys)
is something that we can work on in the future.

## Registering Repositories

Registering repositories can be done via the Web UI or using `kpt`. To register
a GitHub repository `nephio-test-catalog-01` in your personal GitHub account:

```
GITHUB_USERNAME=<your github username>
GITHUB_TOKEN=<GitHub Personal Access Token>

kpt alpha repo register \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  https://github.com/${GITHUB_USERNAME}/nephio-test-catalog-01.git
```

When registering a *deployment* repository for the Workload Cluster that we are going to [create later](#creating-a-workload-cluster). We use the same command, except it
must include the `--deployment` flag:
```
GITHUB_USERNAME=<your github username>
GITHUB_TOKEN=<GitHub Personal Access Token>

kpt alpha repo register \
  --deployment \
  --namespace default \
  --repo-basic-username=${GITHUB_USERNAME} \
  --repo-basic-password=${GITHUB_TOKEN} \
  https://github.com/${GITHUB_USERNAME}/nephio-edge-cluster-01.git
```

Last but not least we also register a "blueprint" repository that contains the sample packages we will use in this PoC.
```
kpt alpha repo register \
  --namespace default \
  https://github.com/nephio-project/nephio-packages.git
```

You should be able to list the three registered repositories.
```
kpt alpha repo get
NAME                     TYPE   CONTENT   DEPLOYMENT   READY   ADDRESS
nephio-edge-cluster-01   git    Package   true         True    https://github.com/GITHUB_USERNAME/nephio-edge-cluster-01
nephio-packages          git    Package                True    https://github.com/nephio-project/nephio-packages.git
nephio-test-catalog-01   git    Package                True    https://github.com/GITHUB_USERNAME/nephio-test-catalog-01.git
```

It is also possible to set a different branch and directory for packages within
the repository; see `kpt alpha repo register --help` for more.

## Creating a Workload Cluster

Workload clusters are  those clusters that do not contain the Nephio system itself,
but instead are intended to run the workloads deployed via Nephio.

```
# Create a workload cluster
gcloud container clusters create-auto --region us-central1 nephio-edge-cluster-01
# This will take a few minutes
# Once it returns, configure kubectl with the credentials for the cluster
gcloud container clusters get-credentials --region us-central1 nephio-edge-cluster-01

# See the nodes in your new cluster
kubectl get nodes
NAME                                                  STATUS   ROLES    AGE     VERSION
gk3-nephio-edge-cluster--default-pool-32693aa1-2d8k   Ready    <none>   1m26s   v1.22.10-gke.600
gk3-nephio-edge-cluster--default-pool-8a128e1b-cl5l   Ready    <none>   1m26s   v1.22.10-gke.600
```

## Installing Config Sync in Workload Clusters

Workload clusters must run Config Sync to get their workloads from their deployment repositories.

The package
[nephio-configsync](https://github.com/nephio-project/nephio-packages/tree/main/nephio-configsync)
is intended for installing Config Sync in those clusters. To use this package,
you will need to update the RootSync resource with the repository and
authentication needed for your environment. 

You can use kpt functions to update the RootSync resource - or you can simply edit the file with your text editor. 
The function method is amenable to automation. You can inject the `GITHUB_USERNAME` environment variable and automate the process as illustrated below.

On the other hand, you could update the resource manually. You can pull the package, edit it with your GitHub username, and then push it to your private catalog. 

In fact, there is no reason you can't just change the resources as you wish, directly.
The Configuration as Data approach is open to the mindset of "you can edit the config either via code or manually". 

Example:
```
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-packages.git/nephio-configsync nephio-edge-cluster-01
# Replace the repository name in rootsync.yaml with the $GITHUB_USERNAME
kpt fn eval nephio-edge-cluster-01 \
  --save \
  --type mutator \
  --image gcr.io/kpt-fn/search-replace:v0.2.0 \
  -- by-path=spec.git.repo by-value-regex='https://github.com/[a-zA-Z0-9-]+/(.*)' \
  put-value="https://github.com/${GITHUB_USERNAME}/\${1}"
kpt fn render nephio-edge-cluster-01
kpt live init nephio-edge-cluster-01
kpt live apply nephio-edge-cluster-01 --reconcile-timeout=15m --output=table

# Verify that the config sync pods are running
kubectl get pods --all-namespaces | grep config
config-management-monitoring   otel-collector-5b8d5d8d8f-8qs2k                                 1/1     Running   0          1m
config-management-system       config-management-operator-6d7867f9f5-xb9cn                     1/1     Running   0          3m
config-management-system       reconciler-manager-c6c8cf7f6-9x7vr                              2/2     Running   0          1m
config-management-system       root-reconciler-nephio-workload-cluster-sync-679dd89788-mkl8x   4/4     Running   0          1m
```

## Deploying a Package Workload
We have two options to provision a sample workload.
  * Use the Nephio Web UI in case you followed the steps in [Installing the Nephio Web UI](#installing-the-web-ui) above, or
  * Use `kpt` to provision the workload from the command line.

[Installing the Nephio Web UI](#installing-the-web-ui) (as described above) was optional.
Therefore, we will use `kpt` to provision the workload.

Example:
```
# Important! We need to set our Kubernetes context back to the nephio cluster first!
kubectl config get-contexts | grep nephio
* gke_nephio-poc_us-central1_nephio-edge-cluster-01   gke_nephio-poc_us-central1_nephio-edge-cluster-01   gke_nephio-poc_us-central1_nephio-edge-cluster-01
  gke_nephio-poc_us-central1_nephio                   gke_nephio-poc_us-central1_nephio                   gke_nephio-poc_us-central1_nephio                       

# Your contexts will be named slightly differently. Make sure you adjust the following command accordingly.
kubectl config use-context gke_nephio-poc_us-central1_nephio

# Get the remote packages available from the registered repositories (output trunkated to focus on coredns-caching package).
kpt alpha rpkg get
NAME                                                              PACKAGE                  REVISION   LATEST   LIFECYCLE   REPOSITORY
nephio-packages-e01d890d4c85fc62299d956829ffe948d712bd76          coredns-caching          main       false    Published   nephio-packages
nephio-packages-edfea244e9255e476de3dcc00b56003104f1d4cd          coredns-caching          v1         true     Published   nephio-packages
...

# Clone the latest coredns-caching revision from the blueprint catalog repo to the deployment repo.
kpt alpha rpkg clone nephio-packages-edfea244e9255e476de3dcc00b56003104f1d4cd dnscache --repository nephio-edge-cluster-01 -n default

# The package is now ready in the deployment repo in the Draft state.
kpt alpha rpkg get
NAME                                                              PACKAGE                  REVISION   LATEST   LIFECYCLE   REPOSITORY
nephio-edge-cluster-01-ffc042a02460c770a3678f4a6b9e3664f9f38983   dnscache                 v1         false    Draft       nephio-edge-cluster-01
nephio-packages-e01d890d4c85fc62299d956829ffe948d712bd76          coredns-caching          main       false    Published   nephio-packages
nephio-packages-edfea244e9255e476de3dcc00b56003104f1d4cd          coredns-caching          v1         true     Published   nephio-packages

# Propose the package
kpt alpha rpkg propose nephio-edge-cluster-01-ffc042a02460c770a3678f4a6b9e3664f9f38983 -n default

# Approve the package (publish it)
kpt alpha rpkg approve nephio-edge-cluster-01-ffc042a02460c770a3678f4a6b9e3664f9f38983 -n default

# Check that the package has been deployed to your Workload Cluster (remember to switch the context back to the workload cluster first)
kubectl config use-context gke_nephio-poc_us-central1_nephio-edge-cluster-01 
kubectl get pods -n dnscache
NAME                               READY   STATUS    RESTARTS   AGE
coredns-caching-757c486c84-xw2gt   1/1     Running   0          3m57s
```
