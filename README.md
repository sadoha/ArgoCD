# Getting Started

!!! tip
    This guide assumes you have a grounding in the tools that Argo CD is based on.

## Requirements

* Installed [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) command-line tool.
* Have a [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file (default location is `~/.kube/config`).

## 1. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This will create a new namespace, `argocd`, where Argo CD services and application resources will live.

!!! warning
    The installation manifests include `ClusterRoleBinding` resources that reference `argocd` namespace. If you are installing Argo CD into a different
    namespace then make sure to update the namespace reference.

## 2. Download Argo CD CLI

Download the latest Argo CD version from [https://github.com/argoproj/argo-cd/releases/latest](https://github.com/argoproj/argo-cd/releases/latest).
Also available in Mac, Linux and WSL Homebrew:

```bash
brew install argocd
```

## 3. Access The Argo CD API Server

By default, the Argo CD API server is not exposed with an external IP. To access the API server,
choose one of the following techniques to expose the Argo CD API server:

### Service Type Load Balancer
Change the argocd-server service type to `LoadBalancer`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Port Forwarding
Kubectl port-forwarding can also be used to connect to the API server without exposing the service.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

The API server can then be accessed using https://localhost:8080


## 4. Login Using The CLI

The initial password for the `admin` account is auto-generated and stored as
clear text in the field `password` in a secret named `argocd-initial-admin-secret`
in your Argo CD installation namespace. You can simply retrieve this password
using the `argocd` CLI:

```bash
argocd admin initial-password -n argocd
```

!!! warning
    You should delete the `argocd-initial-admin-secret` from the Argo CD
    namespace once you changed the password. The secret serves no other
    purpose than to store the initially generated password in clear and can
    safely be deleted at any time. It will be re-created on demand by Argo CD
    if a new admin password must be re-generated.

Using the username `admin` and the password from above, login to Argo CD's IP or hostname:

```bash
argocd login <ARGOCD_SERVER> 

For instance:
argocd login 127.0.0.1:8080
```

Change the password using the command:

```bash
argocd account update-password
```

## 5. Register A Cluster To Deploy Apps To (Optional)

This step registers a cluster's credentials to Argo CD, and is only necessary when deploying to
an external cluster. When deploying internally (to the same cluster that Argo CD is running in),
https://kubernetes.default.svc should be used as the application's K8s API server address.

First list all clusters contexts in your current kubeconfig:
```bash
kubectl config get-contexts -o name
```

Choose a context name from the list and supply it to `argocd cluster add CONTEXTNAME`. For example,
for docker-desktop context, run:
```bash
argocd cluster add docker-desktop
```

The above command installs a ServiceAccount (`argocd-manager`), into the kube-system namespace of
that kubectl context, and binds the service account to an admin-level ClusterRole. Argo CD uses this
service account token to perform its management tasks (i.e. deploy/monitoring).

!!! note
    The rules of the `argocd-manager-role` role can be modified such that it only has `create`, `update`, `patch`, `delete` privileges to a limited set of namespaces, groups, kinds.
    However `get`, `list`, `watch` privileges are required at the cluster-scope for Argo CD to function.

## 6. Preparation to deploy environments

!!! note 
    Before deployment applications, the ArgoCD project should be installed.

In this case, The project name is the same as the environment name. Please take a look at the example below to understand which naming for environments we use:
```bash
100 = dev
200 = test
300 = prod
```

The structure of catalogs:
```bash
.
├── project-100.yaml
├── project-200.yaml
└── project-300.yaml
```

Deploy ArgoCD projects
```bash
kubectl apply -f ./option1/projects
```

Remove ArgoCD projects
```bash
kubectl delete -f ./option1/projects
```

## 7. Deploy the environment using option1

The first configuration option is more for understanding how to configure base applications, override Helm Chart values, set schedulers etc. You can deploy this option to familiarize yourself with the ArgoCD configuration. 

The structure of catalogs:
```bash
.
├── applications
│   ├── 100
│   │   ├── apps
│   │   │   ├── proj-100-app1.yaml
│   │   │   ├── proj-100-app2.yaml
│   │   │   └── proj-100-app3.yaml
│   │   └── proj-100-all-apps.yaml
│   ├── 200
│   │   ├── apps
│   │   │   └── proj-200-app1.yaml
│   │   └── proj-200-all-apps.yaml
│   └── 300
│       ├── apps
│       │   └── proj-300-app1.yaml
│       └── proj-300-all-apps.yaml
└── applicationsets
    └── all-proj-appset.yaml
```

Deploy all environmens:
```bash
kubectl apply -f ./option1/applicationsets/all-proj-appset.yaml
```

Remove all environments:
```bash
kubectl delete -f ./option1/applicationsets/all-proj-appset.yaml
```

Deploy applications per environment (100,200,300):
```bash
kubectl apply -f ./option1/applications/100/proj-100-all-apps.yaml
kubectl apply -f ./option1/applications/200/proj-200-all-apps.yaml
kubectl apply -f ./option1/applications/300/proj-300-all-apps.yaml
```

Remove applications per environment (100,200,300):
```bash
kubectl delete -f ./option1/applications/100/proj-100-all-apps.yaml
kubectl delete -f ./option1/applications/200/proj-200-all-apps.yaml
kubectl delete -f ./option1/applications/300/proj-300-all-apps.yaml
```

## 8. Deploy the environment using option2

The main idea of the option2 is to separate configuration between two repositories, the first one is a repository with ArgoCD configs for deployment applications, and the second one is for configuration of application and Helm Chart values files. In this case, a development team can manage the version of an application, change Helm Chart values, and update the configuration of an application, they will do it in a repository with code. I don't want to dive deeper into the development process, different branches and approaches, because all of than are individual to the project.

The structure of catalogs:
```bash
.
├── applications # This catalog can be located on the development side, in a repository with a code of application
│   └── values
│       ├── app1
│       │   ├── 100
│       │   │   └── values.yaml
│       │   ├── 200
│       │   │   └── values.yaml
│       │   └── 300
│       │       └── values.yaml
│       └── app2
│           ├── 100
│           │   └── values.yaml
│           ├── 200
│           │   └── values.yaml
│           └── 300
│               └── values.yaml
└── applicationsets # This catalog should be in a repository where the ArgoCD configuration is located
    └── proj-appset.yaml
```

Deploy the environment:
```bash
kubectl apply -f ./option2/applicationsets/proj-appset.yaml
```

Remove the environment:
```bash
kubectl delete -f ./option2/applicationsets/proj-appset.yaml
```

## 8. Deploy the environment using option3

This configuration is more appropriate to use in a product. It is a draft, and this configuration neads fine tuning, but, here you can see an example, how to manage the architecture of directories for a projects and environments. This environments can be deployed by three options, as full environment with databases and application, as only applications or infrastructure apps only. I was done to provide a possibility to jangle the environment configuration in a testing and education goals. 

The structure of catalogs:
```bash
./option3
└── argocd
    ├── applicationsets
    │   └── appset-generator-git-all-environments.yaml
    ├── common
    │   ├── infrastructure
    │   │   └── kube-prometheus-stack
    │   │       └── values.yaml
    │   └── variables
    │       └── values.yaml
    ├── environments
    │   ├── 100
    │   │   ├── applicationsets
    │   │   │   ├── appset-generator-git-list-apps.yaml
    │   │   │   └── appset-generator-git-list-infra-apps.yaml
    │   │   ├── infrastructure
    │   │   │   ├── etcd
    │   │   │   │   └── values.yaml
    │   │   │   └── mongodb
    │   │   │       └── values.yaml
    │   │   ├── products
    │   │   │   ├── product1
    │   │   │   │   ├── app1
    │   │   │   │   │   └── values.yaml
    │   │   │   │   └── app2
    │   │   │   │       └── values.yaml
    │   │   │   └── product2
    │   │   │       ├── app1
    │   │   │       │   └── values.yaml
    │   │   │       └── app2
    │   │   │           └── values.yaml
    │   │   └── variables
    │   │       └── values.yaml
    │   └── 200
    │       ├── applicationsets
    │       │   ├── appset-generator-git-list-apps.yaml
    │       │   └── appset-generator-git-list-infra-apps.yaml
    │       ├── infrastructure
    │       │   ├── etcd
    │       │   │   └── values.yaml
    │       │   └── mongodb
    │       │       └── values.yaml
    │       ├── products
    │       │   ├── product1
    │       │   │   ├── app1
    │       │   │   │   └── values.yaml
    │       │   │   └── app2
    │       │   │       └── values.yaml
    │       │   └── product2
    │       │       ├── app1
    │       │       │   └── values.yaml
    │       │       └── app2
    │       │           └── values.yaml
    │       └── variables
    │           └── values.yaml
    └── projects
        └── project.yaml
```

Deploy the environment (option 1):
```bash
kubectl apply -f ./option3/argocd/applicationsets/appset-generator-git-all-environments.yaml
```

Remove the environment (option 1):
```bash
kubectl delete -f ./option3/argocd/applicationsets/appset-generator-git-all-environments.yaml
```

Deploy the environment (option 2):
```bash
kubectl apply -f ./option3/argocd/projects/project.yaml
kubectl apply -f ./option3/argocd/environments/100/applicationsets/appset-generator-git-list-apps.yaml
kubectl apply -f ./option3/argocd/environments/200/applicationsets/appset-generator-git-list-apps.yaml
```

Remove the environment (option 2):
```bash
kubectl delete -f ./option3/argocd/environments/100/applicationsets/appset-generator-git-list-apps.yaml
kubectl delete -f ./option3/argocd/environments/200/applicationsets/appset-generator-git-list-apps.yaml
kubectl delete -f ./option3/argocd/projects/project.yaml
```

Deploy the environment (option 3):
```bash
kubectl apply -f ./option3/argocd/projects/project.yaml
kubectl apply -f ./option3/argocd/environments/100/applicationsets/appset-generator-git-list-infra-apps.yaml
kubectl apply -f ./option3/argocd/environments/200/applicationsets/appset-generator-git-list-infra-apps.yaml
```

Remove the environment (option 3):
```bash
kubectl delete -f ./option3/argocd/environments/100/applicationsets/appset-generator-git-list-infra-apps.yaml
kubectl delete -f ./option3/argocd/environments/200/applicationsets/appset-generator-git-list-infra-apps.yaml
kubectl delete -f ./option3/argocd/projects/project.yaml
```

