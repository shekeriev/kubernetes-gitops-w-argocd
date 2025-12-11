# Demo Script

The demo is planned and organized in a way that makes it easy to follow in a local environment. Of course, it can also be executed against a cloud-based environment.

*Note 1:  It is expected that the workstation already has **kubectl** and **helm** installed. If not, they could be installed by following the steps outlined here (<https://kubernetes.io/docs/tasks/tools/#kubectl>) for **kubectl** and here (<https://github.com/helm/helm/releases>) for **helm**.*

*Note 2: It is expected that on the workstation there is a context defined for the target cluster and it is set as default.*

## Requirements

It is expected that there is a Load Balancer (assumed **MetalLB**), a local git-based solution (assumed **Gitea**), and a CI/CD solution (assumed **Jenkins**). In addition, a read-write access to a container registry (assumed **Docker Hub**) is expected.

Detailed explanation on how to prepare an evironment like the one assumed and used, check the **Preparation** document, available [here](../preparation/preparation.md).

## Demo Steps

Let's assume that we are working in a folder named **demo**. All paths that will follow are relative to this folder.

In addition, make sure you cloned the two imported repositories respectively to **demo/gitops-app** and **demo/gitops-app-infra**.

One more thing. If you decided to follow exactly all the steps and are using **Gitea**, you could export the following two environment variables upfront:

```bash
export $GITEA_IP=ip-address
export $GITEA_USER=user-name
```

*Make sure you substituted **ip-address** and **user-name** with the correct values.*

This way, all the commands that interact with a **Gitea** endpoint will match your situation without further adjustment.

### Without GitOps (CI/CD pipeline)

Let's see how the things happen when we do not have **GitOps** implemented and we are using a CI/CD pipeline to deploy our application.

Now, we should refer to a tool like **Jenkins**.

Go to **Jenkins UI** and start the **pipeline-cicd** pipeline.

After a while our application should be deployed

```bash
kubectl get pods,svc
```

Get the app URL

```bash
minikube -p gitops service gitops-app-svc --url
```

Or with

```bash
kubectl get svc gitops-app-svc
```

And visit it in a browser tab.

Nice. Now, let's do a change. For example, add some text to the **gitops-app/app/index.php** file.

Then stage, commit and push the changes to the repository

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "application change 1"
```

```bash
git push
```

Now, return to **Jenkins UI** and start the **pipeline-cicd** pipeline again.

*Note that we are starting it manually for the sake of simplicity. In the real life it would be configured to trigger automatically via a webhook or by polling the repository periodically for changes.*

After a while, the new version of our application will be deployed to the cluster.

Check the status of the components

```bash
kubectl get pods,svc
```

And re-visit the application.

Now, if we do a direct change in the running configuration of the application, there won't be anything that will act and revert it.

Let's go ahead and prove this by changing, for example, the **number of replicas** to **5**

```bash
kubectl scale deployment gitops-app --replicas=5
```

Check that the changes are there

```bash
kubectl get pods,svc
```

Again, the cluster won't take any action on its own to revert your changes because it accepts that if we changed something, it is because we know what we are doing. ;)

Let's clean up by removing our application

```bash
kubectl delete deployment gitops-app
```

```bash
kubectl delete service gitops-app-svc
```

Now, we are ready to move forward and dive (at least the tip of our toes) into the world of GitOps.

### ArgoCD

#### Installation

Create the namespace

```bash
kubectl create namespace argocd
```

And install it (together with the UI)

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Monitor the progress with

```bash
kubectl get pods -n argocd --watch
```

Once done, you can patch the service to **NodePort** or **LoadBalancer** for easier access

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Let's go for the second option - **LoadBalancer**.

Then check the services

```bash
kubectl get svc -n argocd
```

#### CLI Installation

On a **Linux** distribution you could download the ***latest version*** of the CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Or the ***latest stable*** version with

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
```

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
```

Whichever version you downloaded, you can install it with

```bash
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

And then remove the leftovers

```bash
rm argocd-linux-amd64
```

For other OSes, download the appropriate binary from here: <https://github.com/argoproj/argo-cd/releases/latest>

Once done, you can check the CLI is working

```bash
argocd version --client
```

Should you want command completion, you can enable it with (for Bash)

```bash
. <(argocd completion bash)
```

#### Admin Credentials

Get the initial **admin** password

```bash
argocd admin initial-password -n argocd
```

And use it to login

```bash
argocd login ARGOCD-IP
```

*You must substitute **ARGOCD-IP** with your value.*

Now we can check both the server and CLI versions

```bash
argocd version
```

Next we can change the **admin** password

```bash
argocd account update-password
```

You can use the new credentials to log into the UI and explore.

#### Clusters

To check the list of registered clusters, execute

```bash
argocd cluster list
```

*The next few lines are for informative purposes only. Feel free to skip them.*

Should we want to add a cluster (besides the **in-cluster**), we can start from the available contexts

```bash
kubectl config get-contexts -o name
```

And then add one of them, for example

```bash
argocd cluster add kubernetes-admin@kubernetes
```

#### Project and Applications

We can ask for the list of defined projects

```bash
argocd proj list
```

On a new installation, only one should appear - the **default** project.

In the same manner, we can ask for the list of applications

```bash
argocd app list
```

None should be returned (for now).

#### Create Application (manifests)

Let's register an application for ArgoCD to manage.

Of course, we can do it in the UI, but on the command line is more interesting.

Execute the following command

```bash
argocd app create gitops-app --repo http://$GITEA_IP:3000/$GITEA_USER/gitops-app-infra --path manifests --dest-server https://kubernetes.default.svc --dest-namespace default --label purpose=gitops-demo --label apptype=yaml
```

*You must either substitute **$GITEA_IP** and **$GITEA_USER** with your values or make sure that you have exported upfront those environment variables with correct values.*

(Skip this one for now) Alternatively, we can enable auto sync and self-heal during creation (including namespace autocreation)

```bash
argocd app create gitops-app-yaml --repo http://$GITEA_IP:3000/$GITEA_USER/gitops-app-infra --path manifests --dest-server https://kubernetes.default.svc --dest-namespace argo-app-yaml --sync-policy auto --self-heal --sync-option CreateNamespace=true --label purpose=gitops-demo --label apptype=yaml
```

*You must either substitute **$GITEA_IP** and **$GITEA_USER** with your values or make sure that you have exported upfront those environment variables with correct values.*

Note that if we want to use annotations, the above (the last part of the command) will change to `--annotations purpose=gitops-demo,apptype=yaml`

However, as with Kubernetes, annotations and labels serve different purposes. Annotations are for metadata, while labels allows us to use them for filtering/narrowing down a list of resources.

Before we continue, do you remember that we mentioned ***declarative*** during the **ArgoCD** introduction?

So far, we did not see anything that reminds of this.

Let's tackle this issue.

The above application, could be declared in **ArgoCD** following the declarative approach by using a manifest file (for example, `app.yaml`) like this

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://GITEA_IP:3000/GITEA_USER/gitops-app-infra
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

*You must substitute **GITEA_IP** and **GITEA_USER** with your values.*

(Skip this one for now) And send it to ArgoCD via command like this

```bash
argocd app create --file app.yaml
```

No matter how we created our first application in **ArgoCD**, we could ask for the list of registered applications

```bash
argocd app list
```

And then for details about the gitops-app

```bash
argocd app get gitops-app
```

Explore the information.

Pay attention to this line URL: <https://ARGOCD-IP/applications/gitops-app>

Open a browser and visit the above.

Use the admin credentials.

In the UI, we have the same - **the application is out of sync**.

We can sync it either from the UI by going to **Applications** and clicking either **SYNC APPS** for all of the apps or just **SYNC** for particular app (the one that we have in our case) or by executing the following command

```bash
argocd app sync gitops-app
```

No matter the way you triggered synchronization, the app's resources will be deployed and this will be reflected both in the UI and on the CLI

```bash
argocd app list
```

```bash
argocd cluster list
```

We can explore the resources of the application (it is in the **default** namespace)

```bash
kubectl get all
```

Get the app URL and visit it

```bash
minikube -p gitops service gitops-app-svc --url
```

Let's delete one of application's resources and see what will happen. For example, the deployment

```bash
kubectl delete deployment gitops-app
```

After a while the deployment and everything managed by it will be gone

```bash
kubectl get all
```

Switch to the UI and check. It will detect that there is a change and will mark the app as **OutOfSync**.

Now, because our application is not configured with automatic sync, it won't heal itself.

We can enable the automatic sync again from the UI (**Application** > ***particular application*** > **Details** > **Sync Policy** and enable **Self Heal**) or by executing this

```bash
argocd app set gitops-app --sync-policy auto --self-heal
```

Even with this, when there is a change in the repository, it may take up to ***3 minutes***. More information:

- <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-sync-semantics>

- <https://argo-cd.readthedocs.io/en/stable/faq/#how-often-does-argo-cd-check-for-changes-to-my-git-or-helm-repository>

Self-heal, when configuration drift is detected, will take considerably less time - around ***5 seconds***. Details here: <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-sync-semantics>

Go ahead and enable self-healing either in the UI or on the CLI.

Return in the UI and watch the self-healing taking place.

Now, besides the information available in the UI, we can use a set of commands to explore our application and its state.

We can ask for the app's logs

```bash
argocd app logs gitops-app
```

We can check app's revision

```bash
argocd app history gitops-app
```

We can ask for application details (data), but first refresh it

```bash
argocd app get gitops-app --refresh
```

We can ask for application details (data) and operation

```bash
argocd app get gitops-app --show-operation
```

We can ask for application details (data) and draw a tree showing its components

```bash
argocd app get gitops-app --output tree
```

And we could combine the additional blocks of information, for example, the operation and tree of components

```bash
argocd app get gitops-app --show-operation --output tree
```

Of course, all the above plus more could be seen easily in the UI.

You could go ahead and change the application. For example, add some text to the **gitops-app/app/index.php** file.

Then stage, commit and push the changes to the repository

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "application change 2"
```

```bash
git push
```

Now, return to **Jenkins UI** and start the **pipeline-gitops** pipeline.

*Note that we are starting it manually for the sake of simplicity. In the real life it would be configured to trigger automatically via a webhook or by polling the repository periodically for changes.*

After a while, the new version of our application will be deployed to the cluster because the **GitOps operator** (***ArgoCD*** in our case) will capture that there is a change in the desired state and will make sure that the actual state is matching the desired one.

Once the pipeline completes, you can observe the changes that will take place in the cluster.

After ***up to 3 minutes*** **ArgoCD** will sync the application with the current state in **Git**.

Of course, if we do not want to wait, we can execute the synchronization manually either from the UI or on the command line

```bash
argocd app sync gitops-app
```

Check the application's history, if you triggered the **GitOps** pipeline, either in the UI by going to **Applications** > ***particular application*** > **HISTORY AND ROLLBACK** or by executing the following

```bash
argocd app history gitops-app
```

We can continue poking around with our application or go ahead and try a different type of application.

#### Create Application (helm)

To create a **Helm**-based application on the command line, we can execute the following

```bash
argocd app create gitops-app-helm --repo http://$GITEA_IP:3000/$GITEA_USER/gitops-app-infra --path helm/gitops-app --dest-namespace argo-app-helm --dest-server https://kubernetes.default.svc --sync-policy auto --self-heal --sync-option CreateNamespace=true --helm-set replicaCount=2 --label purpose=gitops-demo --label apptype=helm
```

*You must either substitute **$GITEA_IP** and **$GITEA_USER** with your values or make sure that you have exported upfront those environment variables with correct values.*

Now, either use the UI or the ususal commands that you are already aware of, to explore our application and its state.

```bash
argocd app list
```

```bash
argocd app get gitops-app-helm
```

Should you want, you can trigger the **GitOps** pipeline again and observe.

You could go ahead and change the application. For example, add some text to the **gitops-app/app/index.php file**.

Then stage, commit and push the changes to the repository

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "application change 3"
```

```bash
git push
```

Now, return to **Jenkins UI** and start the **pipeline-gitops** pipeline again.

*Note that we are starting it manually for the sake of simplicity. In the real life it would be configured to trigger automatically via a webhook or by polling the repository periodically for changes.*

After a while, the new version of our application will be deployed to the cluster because the **GitOps operator** (***ArgoCD*** in our case) will capture that there is a change in the desired state and will make sure that the actual state is matching the desired one.

Once the pipeline completes, you can observe the changes that will take place in the cluster.

After ***up to 3 minutes*** **ArgoCD** will sync the application with the current state in **Git**.

Of course, if we do not want to wait, we can execute the synchronization manually either from the UI or on the command line

```bash
argocd app sync gitops-app-helm
```

Check the application's history (if you triggered the **GitOps** pipeline)

```bash
argocd app history gitops-app-helm
```

Okay, but how could we check the outcome?

If we ask for the service's URL with the usual command

```bash
minikube -p gitops service gitops-app-helm-svc -n argo-app-helm --url
```

We won't see anything because the service type is set to **ClusterIP** (in the **values.yaml** file).

How could we address this?

We could do a PR against the infrastructure repository and see what will happen.

Open the **gitops-app-infra/helm/gitops-app/values.yaml** file for editing.

And change the service type to **NodePort**.

Then save the changes, stage, commit and push them to the repository

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Change service type to NodePort"
```

```bash
git push
```

Now, return to the UI and watch what will happen.

Once, the synchronization is done, get the service URL again

```bash
minikube -p gitops service gitops-app-helm-svc -n argo-app-helm --url
```

And visit the application in a browser tab.

We can continue poking around with our application or go ahead and try a different type of application.

#### Create Application (kustomize)

To create a **kustomize**-based application on the command line, we can execute the following

```bash
argocd app create gitops-app-kust --repo http://$GITEA_IP:3000/$GITEA_USER/gitops-app-infra --path kustomize/overlays/stg --dest-namespace argo-app-kust --dest-server https://kubernetes.default.svc --sync-policy auto --self-heal --sync-option CreateNamespace=true --kustomize-namespace argo-app-kust --label purpose=gitops-demo --label apptype=kust
```

*You must either substitute **$GITEA_IP** and **$GITEA_USER** with your values or make sure that you have exported upfront those environment variables with correct values.*

Now, either use the UI or the ususal commands that you are already aware of, to explore our application and its state.

```bash
argocd app list
```

```bash
argocd app get gitops-app-kust
```

You can always ask for the service and check the application in a browser tab

```bash
minikube -p gitops service gitops-app-svc -n argo-app-kust --url
```

Should you want, you can trigger the **GitOps** pipeline again and observe.

You could go ahead and change the application. For example, add some text to the **gitops-app/app/index.php file**.

Then stage, commit and push the changes to the repository

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "application change 4"
```

```bash
git push
```

Now, return to **Jenkins UI** and start the **pipeline-gitops** pipeline again.

*Note that we are starting it manually for the sake of simplicity. In the real life it would be configured to trigger automatically via a webhook or by polling the repository periodically for changes.*

After a while, the new version of our application will be deployed to the cluster because the **GitOps operator** (***ArgoCD*** in our case) will capture that there is a change in the desired state and will make sure that the actual state is matching the desired one.

Once the pipeline completes, you can observe the changes that will take place in the cluster.

After ***up to 3 minutes*** **ArgoCD** will sync the application with the current state in **Git**.

Of course, if we do not want to wait, we can execute the synchronization manually either from the UI or on the command line

```bash
argocd app sync gitops-app-kust
```

Check if the application reflected the changes you made.

Finally, check the application's history (if you triggered the **GitOps** pipeline)

```bash
argocd app history gitops-app-kust
```

#### Complete Removal

First, we can explore and delete the applications either one-by-one or in batches.

Get a list of all applications

```bash
argocd app list
```

Or filter the list by label

```bash
argocd app list -l apptype=yaml
```

(Skip this) Delete single application

```bash
argocd app delete gitops-app
```

(Skip this) Or delete multiple at once

```bash
argocd app delete gitops-app-helm gitops-app-kust gitops-app-yaml
```

(Use this) Or delete all that have particular label

```bash
argocd app delete -l purpose=gitops-demo
```

Finally, we can uninstall the **ArgoCD** suite

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

And remove the namespace as well

```bash
kubectl delete namespace argocd
```
