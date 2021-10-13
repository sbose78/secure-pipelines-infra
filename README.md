# OpenShift Pipelines with Supply Chain Security

What would a secure supply chain in a managed CI/Build service powered by OpenShift Pipelines look like ?

A user of this service would be able to do the following:

* Login with Github based on her membership to the https://github.com/redhat-developer org.
* Build a container image from source and push it to docker.io .
* Sign and attest the image and push the same to docker.io
* Upload a transparency log of the build process in a pre-defined format to https://rekor.sigstore.dev/ .

This is a GitOps repository of the setup that can be re-created in ~3 minutes on an OpenShift 4.8 cluster. 

Note, this setup has external dependencies, namely,

* Secret management using Akeyless.io.
* Creation of Github client ID and secrets - stored in Akeyless.io Vault
* Creation of [Cosign](https://github.com/sigstore/cosign) keys -  - stored in Akeyless.io Vault


## Components

### Status

**Status** : Successfully logged in using Github, built an image using the Kaniko Tekton Task, pushed signatures/attestations to Dockerhub and pushed transparency log to https://rekor.sigstore.dev/


### Stack

Following components would need to work together in the hypothetical managed service. 

| Component  | Status |
| ------------- | ------------- |
| OpenShift Pipelines  | Installed in-cluster |
| Tekon Chains | Installed in-cluster |
| External Secrets | Used with akeyless.io
| Tekton Results | TODO |
| Shipwright  | Installed in-cluster  |
| Rekor | TODO, using https://rekor.sigstore.dev/ for now |
| Policy Management | TODO 


## User guide

1. Create a new Project.

2. Create a `Secret` named `my-docker-credentials` with your docker.io credentials

```
REGISTRY_SERVER=https://index.docker.io/v1/ REGISTRY_USER=<your_registry_user> REGISTRY_PASSWORD=<your_registry_password>
kubectl create secret docker-registry push-secret \
    --docker-server=$REGISTRY_SERVER \
    --docker-username=$REGISTRY_USER \
    --docker-password=$REGISTRY_PASSWORD  \
    --docker-email=<your_email>
```

3. Link the `Secret` to your `pipeline` service account so that Tekton Chains knows where to push artifact signatures/attestations to.

```
oc patch serviceaccount pipeline \
  -p "{\"imagePullSecrets\": [{\"name\": \"my-docker-credentials\"}]}" -n $NAMESPACE
```

4. Proceed to building an image using Shipwright or Tekton


### Shipwright Builds

#### 

Define a `Build` by importing the following YAML

```
---
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  name: s2i-nodejs-build
spec:
  source:
    url: https://github.com/shipwright-io/sample-nodejs
    contextDir: source-build/
  strategy:
    name: source-to-image
    kind: ClusterBuildStrategy
  builder:
    image: docker.io/centos/nodejs-10-centos7
  output:
    image: docker.io/foo/bar
    credentials:
      name: my-docker-credentials
```

Kick-off your first `Build` by creating a `BuildRun`

```
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: s2i-nodejs-buildrun-
spec:
  buildRef:
    name: s2i-nodejs-build
```

And that's it, wait for the BuildRun to complete. You should see an image, a signature and an attestation pushed to your registry.


![image](https://user-images.githubusercontent.com/545280/137172847-1827201e-e31a-4f04-ab78-a633149435a5.png)



### Tekton Task Builds

The above could be accomplished by a Tekton TaskRun as well.

1. Link the `Secret` named `my-docker-credentials` to your `pipeline` service account. This somewhat repititive step will go away when an appropriate Task is used.

```
oc patch serviceaccount pipeline \
  -p "{\"Secrets\": [{\"name\": \"my-docker-credentials\"}]}" -n $NAMESPACE
```

2. Import the following YAML to build your image. After a successful run, find the image, the corresponding signature and the attestation in DockerHub : https://raw.githubusercontent.com/sbose78/secure-pipelines-infra/main/examples/generated-shipwright.yaml

Modify the above YAML to replace `IMAGE_URL`'s value with your own.



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

