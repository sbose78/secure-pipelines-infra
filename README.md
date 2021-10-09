# OpenShift Pipelines with Supply Chain Security

## Components

* OpenShift Pipelines
* Tekon Chains
* Secret management

## Getting Started

1. Install OpenShift GitOps

2. Create an Application CR in the `openshift-gitops` namespace to setup the secure pipelines stack including OpenShift Pipelines, Tekton Chains and External Secrets.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pipelines
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: openshift-pipelines
    repoURL: https://github.com/sbose78/managed-tekton
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

3. Configure External Secrets to authenticate with akeyless.io. This assumes that you have already setup the credentials in akeyless.io vault to push to docker.io.

4. Setup a Kanino build by setting up the `Task` in the `user-pipelines` namespace

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-pipelines
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: user-pipelines
    repoURL: https://github.com/sbose78/managed-tekton
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
