# OpenShift Pipelines with Supply Chain Security

What would a secure supply chain in a managed CI/Build service powered by OpenShift Pipelines look like ?

A user of this service would be able to do the following:
* Build a container image from source, 
* Push it to an OCI resgistry
* Sign & attest the image
* Upload a transparency log of the build process.


This is a GitOps repository of the setup that can be re-created in ~3 minutes on an OpenShift 4.8 cluster. 

Note, this setup has external dependencies, namely,

* Secret management using Akeyless.io.
* Creation of Github client ID and secrets - stored in Akeyless.io Vault
* Creation of [Cosign](https://github.com/sigstore/cosign) keys -  - stored in Akeyless.io Vault


## Components

Following components would need to work together in the hypothetical managed service. 


| Component  | Status |
| ------------- | ------------- |
| Shipwright  | TODO  |
| OpenShift Pipelines  | Installed in-cluster |
| External Secrets | Used with akeyless.io
| Tekton Results | TODO |
| Tekon Chains | Installed in-cluster |
| Rekor | TODO, using https://rekor.sigstore.dev/ for now |
| Policy Management | TODO 



## Installation


### Install everything using GitOps!

Create an Application CR in the `openshift-gitops` namespace to setup the secure pipelines stack including OpenShift Pipelines, Tekton Chains, External Secrets, signing keys and Github OAuth Client Secret.

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
    repoURL: https://github.com/sbose78/secure-pipelines-infra
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Secrets management

3. Configure External Secrets to authenticate with akeyless.io. This assumes that you have already setup the credentials in akeyless.io vault to push to docker.io. Set the following environment variables in your external secrets deployment.

```
AKEYLESS_ACCESS_TYPE : access_key
AKEYLESS_ACCESS_ID : <ACCESS ID>
AKEYLESS_ACCESS_TYPE_PARAM: <THE ACTUAL ACCESS KEY>

```

Two `ExternalSecrets` have been defined in this installation process.

* Signing secrets that would be used to create the image signature prior to pushing the same to an OCI registry. The keys used in the [Signing Secret](openshift-pipelines/05-signing-secrets.yaml) must be configured on your Akeyless.io account.

* The Github.com client secret that would be consumed by the `OAuth` CR to configure logging in to OpenShift using Github.com.



### Set up Github login


Enable Github login for your OpenShift instance so that users are able to login with their Github.com account by creating the following CR:

```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - github:
        clientID: SET_YOUR_OWN
        clientSecret:
          name: github-client-secret-cp
        hostname: ''
        organizations:
          - redhat-developer
        teams: []
      mappingMethod: claim
      name: github
      type: GitHub
```

The `secret` named `github-client-secret-cp` referred to in the yaml snippet above should be created prior to this.

