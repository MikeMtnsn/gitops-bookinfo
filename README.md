# Setting up gitops with argocd

## Prereq
There is already two running Tekton pipelines for:
1. productpage
2. details-v2 

You should have those in your `pipeline-$INITIALS` project.

There is already a running argocd - setup via the argocd operator in namespace `arcgocd`.
(For later records you should be able to see the setup [here](https://github.com/ibm-garage-cph/openshift-gitops-101/ ).

### Argocd access
The default argocd you need two key informations about:
1. URL. Retrieve it via the route: `oc get route -n argocd`. Hint you can store this in an ARGOCDURL environment variable.
2. Password. Retrive it via the secret: `oc get secret example-argocd-cluster -o jsonpath='{.data.admin\.password}' -n argocd | base64 -d && echo`. Hint you can store this in an ARGOCDPW environment variable and/or put it into your .secret file.
3. Username will be shared and be `admin`.

### Argocd command line tool
[Installation instructions](https://argoproj.github.io/argo-cd/cli_installation/)

### Argocd login
From web console use `$ARGOCDURL`, and apply username and password when asked.

From command line use `argocd login $ARGOCDURL`....


### Github repos access
You will need a github account to be able to store your configuration of what is going to be deployed. If you do not have a github.com account please create one. And setup an empty repository that you will populate at first, and later let the Tekton CI update it, and ArgoCd take the information and use it for deployment.
You then need to generate a ["personal access token"](https://github.com/settings/tokens). Access needs are `repo`:  . Suggest to store it in `.secrets`.

#### Fork gitops repos
Fork this git repository `https://github.com/ibm-garage-cph/bookinfo-productpage-gitops.git`. Save the URL to $GITOPS_REPO.
Look at the content and if anything needs to be changed (hint: hostnames...) then change it and push back the updates...


### Preparing argocd for your namespace
Argocd also needs to be able to deploy into the workspace being simulated as "production".
First you need to create the empty project name it `bookinfo-prod-$INITIALS`. You also need to ensure it can download container images to it (Add secret and ensure service account uses secret).

```bash
oc new-project bookinfo-prod-$INITIALS
oc get secrets default-icr-io  -n default --export -o yaml | oc apply -n bookinfo-prod-$INITIALS -f -
oc patch serviceaccount default -n bookinfo-prod-$INITIALS -p '{"imagePullSecrets": [{"name": "default-icr-io"}]}'
```

Once it is created, you need to grant access from argocd to be able to deploy updates to it.
```bash
oc adm policy add-cluster-role-to-user cluster-admin -z argocd-application-controller -n argocd
```


# Update tekton to do gitops (CD) prep and only that for prod
Add a step to the pipeline to update the gitops repos with new image to be deployed.

## Define gitops-update
Find your pipeline name space `pipeline-$INITIALS` and define a new task `gitops-task` in there.

```bash
oc project pipelines-$INITIALS
oc apply -f gitops-task.yaml

```

Append a new `params` named `GITOPS_REPO` to your pipeline like below (notice this will change as Tekton evolves for v1alpha1 below is the format). Insert it between `spec` and `resource.

```yaml
spec:
  params:
    - name: GITOPS_REPO
      type: string
  resources:
    - name: git-repo
      type: git
    - name: image
```

Also add the task `gitops-update` as the last step:
```yaml
    - name: gitops-update
      params:
        - name: GITOPS_REPO
          value: $(params.GITOPS_REPO)
      resources:
        inputs:
          - name: source
            resource: git-repo
          - name: image
            resource: image
      runAfter:
        - build-image
      taskRef:
        kind: Task
        name: gitops-task
```

Also the pipeline serviceaccount needs to be able to do updates to github via the token generated before.

Create the secret named `gitops-github`. Do notice the annotations `tekton.dev/git-0`which ensures Tekton uses the secret. Replace GITUSER and GITTOKEN with your username and token in `gitops-github.yaml`

```bash
  oc apply -f gitops-github.yaml
```
Associate the secret to the `pipeline` service account.
```bash
  oc patch serviceaccount pipeline -p '{"secrets": [{"name": "gitops-github"}]}'
```

Last run the pipeline again this time with the added parameter GITOPS_REPO to your list of arguments to `tkn`. Replace git-repo and image with your resource definitions for yoru pipeline.

```bash
  tkn pipeline start continous-integration \
    -r git-repo=productpage-repo \
    -r image=productpage-image \
    -p GITOPS_REPO="$GITOPS_REPO"
```
Move on to create the argocd steps. But do come back and check the result of the pipeline to see it ran without errors.

# Create project in argocd to do CD
This will be described via commandline, but can be applied equivalently from web console.

First define the gitops repository to be used:
```bash

````

Then create the argocd project.

```bash
argocd proj create bookinfo-prod-$INITIALS --orphaned-resources --orphaned-resources-warn --insecure -d https://kubernetes.default.svc,bookinfo-prod-$INITIALS
```

## Create productpage in project
Then create the application productpage inside the argocd project.

Select create projectand use below information:
```
project: bookinfo-prod-$INITIALS
source:
  repoURL: '$GITOPS_REPO'
  path: prod
  targetRevision: HEAD
destination:
  server: 'https://kubernetes.default.svc'
  namespace: bookinfo-prod-$INITIALS
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```


See how it deploys, and correct any errors as needed. (Check exposed URL works)

## create details-v2 in project



### Links
[ArgoCD inspiration](https://github.com/ibm-garage-cph/openshift-gitops-101/)
Inspiration from [cp4app liberty app modernization](https://ibm-cloud-architecture.github.io/cloudpak-for-applications/liberty/liberty-deploy-tekton-argocd/).
