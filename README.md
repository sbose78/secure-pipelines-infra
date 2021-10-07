# OpenShift Pipelines with Supply Chain Security

## Components

* OpenShift Pipelines
* Tekon Chains
* Secret management

## Getting Started

1. Install OpenShift GitOps
2. Create an Application CR in the `openshift-gitops` namespace to setup everything

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
