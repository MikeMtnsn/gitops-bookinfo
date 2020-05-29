# Setting up gitops with argocd

## Optional
### Tekton
There could already be two running Tekton pipelines for:
1. productpage-v2
2. details-v2 

If you have them they should be in your `tekton-$INITIALS` project.

If this is not the case do not worry - you can do this later. This task is possible to perform without.

## Prereq
### Github repos access
You will need a github account to be able to store your configuration of what is going to be deployed. If you do not have a github.com account please create one. 

If you want to let Tekton do updates later on, or you want to make the repository private then you need to generate a ["personal access token"](https://github.com/settings/tokens). Access needs are `repo`:  . Suggest to store it in `.secrets` (or where works for you).

### Argocd instance
There is already a running argocd - setup via the argocd operator in namespace `arcgocd`. (You can see more about this under links).

#### Argocd access information
To access the default argocd you need two key informations. These are:
1. URL. Retrieve it via the route: `oc get route -n argocd`. Hint you can store this in an ARGOCDURL environment variable.
2. Password. Retrive it via the secret: `oc get secret example-argocd-cluster -o jsonpath='{.data.admin\.password}' -n argocd | base64 -d && echo`. Hint you can store this in an ARGOCDPW environment variable and/or put it into your .secret file. We will provide this information to avoid any multi platform issues.
3. Username will be shared and be `admin`.


## Setup argocd project and application...

### Fork gitops repos
Fork this git repository `https://github.com/ibm-garage-cph/gitops-bookinfo`. 
We recommend you make it public for this exercise, however for a real setup this should certainly be a well protected private repository.
Save the URL to $GITOPS_REPO.
Look at the content and if anything needs to be changed (hint: hostnames...) then change it and push back the updates...

### Argocd login
From web console use `$ARGOCDURL`, and apply username and password when asked.

### Preparing argocd for your namespace
Argocd also needs to be able to deploy into the workspace being simulated as "production".
We will allow argocd to run across the cluster for simplicity:
```bash
oc project bookinfo-$INITIALS
oc adm policy add-cluster-role-to-user cluster-admin -z argocd-application-controller -n argocd
oc adm policy add-cluster-role-to-user cluster-admin -z argocd-server -n argocd
```

## Create project in argocd to do CD

### Register your github repository in argocd
Go to "manage your repositiroes..." aka "settings".
Select *Repositories*.

If the repos you want to use does not exist already create it. If it exist skip this step.

Select *CONNECT REPO USING HTTPS*.

Provide input for:
```
Type=GIT
Repository URL=GITOPS_REPO (or https://github.com/ibm-garage-cph/gitops-bookinfo)
username=(if you made it private)
password=(if you made it private)
```
Select *CONNECT*.

### Create your project in argocd
Go to "manage your repositiroes..." aka "settings".
Select *Projects*

Select *NEW PROJECT*

Provide input for:
```
Project name: bookinfo-$INITIALS

Sources: click *add source* select your git repository.
Destinations: bookinfo-$INITIALS

```
Select *CREATE*.


### Create your application in argocd
Go to "manage your applications..." aka "applications.

Select *NEW APP*

Provide input for:
```
Application name: $INITIALS-productpage-v2
Project: bookinfo-$INITIALS
SYNC POLICY: Manual

SOURCE:
Repository URL: GITOPS_REPO (or https://github.com/ibm-garage-cph/gitops-bookinfo)
Revision: HEAD
Path: prod

DESTINATION:
Cluster: in-cluster
Namespace: bookinfo-$INITIALS

Directory:
no changes

```
Select *CREATE*.


## Deploy like a gitops

### Diff
First do an "app diff". How far is your application actually from the gitops repos?

Is it realistci that a sync will succed?

### Sync to prod
Select sync and then synchronize.

WHat happens?

Can you still access your bookinfo page? Is the gateway part of the setup being monitoried by argocd?

### Test and update to prod-v1+
Now lets run some workload to see the new v2 release. Try the following steps.

Select *app details*. Then select *Edit*. Then change path from `prod` to `prod-v1+`. Select *SAVE*.
What happens?

Look at "app diff" again.

Will a sync succed? Try it *sync* then *synchronize*.

Can you still access your bookinfo page? If you load it 10 times will you see the book cover?

### Test and update to prod-v2
Change the path from `prod-v1+` to `prod-v2`.

Repeat the steps from before. What do you expect will be the output?

### Test and update to prod-v2-fail
Change the path from `prod-v2` to `prod-v2-fail`.

Repeat the steps from before. What do you expect will be the output?

Maybe worth going back to `prod-v2` .... or?


### What would it ake to deploy to another cluster?

### What would it ake to deploy to another cloud provider or private cloud?


# Exercise done

### Links
[ArgoCD inspiration](https://github.com/ibm-garage-cph/openshift-gitops-101/)
Inspiration from [cp4app liberty app modernization](https://ibm-cloud-architecture.github.io/cloudpak-for-applications/liberty/liberty-deploy-tekton-argocd/).

#### Argocd command line tool
There is a command line tool for argocd, for convinience it is provided here as well:
[Installation instructions](https://argoproj.github.io/argo-cd/cli_installation/)



# STOP HERE - SKIPPED

## Create a new bookinfo-prod namespace & run tekton
First you need to create the empty project name it `bookinfo-prod-$INITIALS`. You also need to ensure it can download container images to it (Add secret and ensure service account uses secret).

```bash
oc new-project bookinfo-prod-$INITIALS
oc get secrets default-icr-io  -n default --export -o yaml | oc apply -n bookinfo-prod-$INITIALS -f -
oc patch serviceaccount default -n bookinfo-prod-$INITIALS -p '{"imagePullSecrets": [{"name": "default-icr-io"}]}'
```

Once it is created, you need to grant access from argocd to be able to deploy updates to it.
```bash
oc adm policy add-cluster-role-to-user cluster-admin -z argocd-application-controller -n argocd
oc adm policy add-cluster-role-to-user cluster-admin -z argocd-server -n argocd
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
