
# Building a Continuous Deployment GitOps Pipeline to EKS

This tutorial describes how to set up a basic GitOps deployment pipeline with Flux and GitHub actions. It uses the Kubernetes Guestbook application and also adapts a [GitHub Actions pipeline](https://github.com/github-developer/example-actions-flux-eks) originally developed by Jeremy Adams and John Bohannon of GitHub.


## What do we need for GitOps on AWS?

These are the tools that you’ll be working with in order to create a basic GitOps pipeline for application deployments or even for cluster components. If running this in production, you might also need an additional component to scan your base images for added security.


|         Component         	| Implementation                        	| Notes                                                                                                                               	|
|:-------------------------:	|---------------------------------------	|-------------------------------------------------------------------------------------------------------------------------------------	|
| EKS                       	| [EKSctl](https://eksctl.io/)                                	| Managed Kubernetes service.                                                                                                         	|
| Git repo                  	| https://github.com/weaveworks/guestbook-gitops 	| A Git repository containing your application and cluster manifests files.                                                           	|
| Continuous Integration    	| [GitHub Actions](https://github.com/features/actions)                        	| Test and integrate the code - can be anything from CircleCI to GitHub actions.                                                      	|
| Continuous Delivery       	| [Flux version 2](https://toolkit.fluxcd.io/cmd/flux/)                                  	| Cluster <-> repo synchronization                                                                                                    	|
| Container Registry        	| [AWS ECR Container Registry](https://aws.amazon.com/ecr/)            	| Can be any image registry or even a directory.                                                                                      	|
| GitHub secrets management 	| ECR                                   	| Can use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [Vault](https://www.vaultproject.io/).In this case we are using Elastic Container Registry (ECR) which provides resource level  security. 	|



## Before you begin:

You will need to sign up for the following services:

* AWS account  with the ability to create EKS clusters
* GitHub account
* ECR Container Registry (we'll step through this below)
* Kubectl version 1.18 or newer
* [Kustomize](https://github.com/kubernetes-sigs/kustomize)

In this example, you will install the standard [Guestbook sample application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/) to EKS. Then you’ll make a change to the look of the buttons and deploy it to EKS with GitOps.

## Part 1: Fork the Guestbook app repository

You will need a GitHub account for this step.

* Fork and clone the repository: `weaveworks/guestbook-gitops`
* Keep the repo handy as you’ll be adding a deployment key to the repo that Flux requires as well as a few credentials once you have your ECR AWS container registry set up.

## Part 2: Create an EKS cluster:

1. Set up permissions for eksctl:

* Authenticate your cli and ensure your IAM credentials are set up properly before running eksctl:

  * Use at least version 1.17 of the AWS cli
  * Run: `aws configure` to [set up your credentials and the AWS region](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)

* To pull images from the ECR registry, you’ll need to set the correct IAM permissions for the same user that set up your cluster. You can set this from the console or from the command line:

  * The AWS-managed `AmazonEC2ContainerRegistryPowerUser` policy - must be set to `create` and to `push/pull`

2. Stand up the cluster:

* [Download and install](https://github.com/weaveworks/eksctl) or update the command line tool, [eksctl](https://eksctl.io/).
* Run `. <(eksctl completion bash)`
* Run `eksctl create cluster`
* The cluster will take a few minutes to come up.
* Once the cluster creation is completed, run:

`kubectl get pods -A`

You should see something like this:

```
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-284tq             1/1     Running   0          20m
kube-system   aws-node-csvs7             1/1     Running   0          20m
kube-system   coredns-59dfd6b59f-24qzg   1/1     Running   0          27m
kube-system   coredns-59dfd6b59f-knvbj   1/1     Running   0          27m
kube-system   kube-proxy-25mks           1/1     Running   0          20m
kube-system   kube-proxy-nxf4n           1/1     Running   0          20m
```

## Part 2: Create an ECR container registry Account and set up GitHub Actions Workflow

In this section, you will set up an ECR container repository and a mini CI pipeline using GitHub Actions. The actions builds a new container on a `git push`, tags it with the git-sha and then pushes it to the ECR registry; it also updates and commits the image tag change to your kustomize file.  Once the new image is in the repository, Flux notices the new image and then deploys it to the cluster. This entire flow will become more apparent in the next section after you’ve configured Flux.

1. Open an ECR (Elastic Container Registry) account

Open an [ECR account](https://aws.amazon.com/ecr/) in the same region that your cluster is running.  You can call the repository guestbook.

2. Specify secrets for ECR


ECR is an encrypted container repository and as a result any images pulled to and from it need to be authenticated.  You can specify secrets for ECR in the Settings → Secrets tab on your forked guestbook-gitops repository.  These are needed by the GitHub Actions script before it can push the new image to the container registry.

Create the following three GitHub secrets:

```
AWS_ACCOUNT_ID
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```
The secret access key is found on your AWS account ID and is obtainable from the AWS console or by running `aws sts get-caller-identity` and the new user’s key ID and the secret key (found in your `~/.aws/credentials` file).


### Configure the GitHub actions workflow

View the workflow in your repo under .github/workflows/main.yml. Ensure you have the environment variables on lines 16-20 of main.yml set up properly.

```
16 AWS_DEFAULT_REGION: eu-west-1
17 AWS_DEFAULT_OUTPUT: json
18 AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
19 AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
20 AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

Line 16 is the region where your EKS cluster resides. This should match the region in your `~/.aws/config` or the region you specified when you ran `aws configuration`.


## Part 3: Install Flux and start shipping

Flux keeps Kubernetes clusters in sync with configuration kept under source control like Git repositories, and automates updates to that configuration when there is new code to deploy. It is built using Kubernetes' API extension server, and can integrate with Prometheus and other core components of the Kubernetes ecosystem. Flux supports multi-tenancy and syncs an arbitrary number of Git repositories.

In this section, we’ll set up Flux to synchronize changes in the guestbook-gitops repository. This example is a very simple pipeline that demonstrates how to sync one application repo to a single cluster,  but as mentioned, Flux is capable of a lot more than that.  For an overview of all of Flux’s features, refer to [More detail on what’s in Flux v2](https://toolkit.fluxcd.io/#more-detail-on-whats-in-flux).



1. For MacOS users, you can install flux with Homebrew:

`brew install fluxcd/tap/flux`

For other installation options, see [Installing the Flux CLI](https://toolkit.fluxcd.io/get-started/#install-the-flux-cli).

2. Once installed, check that your EKS cluster satisfies the prerequisites:

`flux check --pre`

If successful, tt returns something similar to this:
```
► checking prerequisites
✔ kubectl 1.19.3 >=1.18.0
✔ Kubernetes 1.17.9-eks-a84824 >=1.16.0
✔ prerequisites checks passed
```

Flux supports synchronizing manifests in a single directory, but when you have a lot of YAML it is more efficient to use Kustomize to manage them. For the Guestbook example, all of the manifests were copied into a deploy directory and a kustomization file was added.  For this example, the kustomization file contains a `newTag` directive for the frontend images section of your deployment manifest:
```
images:
- name: frontend
  newName:guestbook
  newTag: new
```
As mentioned above, the Github Actions script updates the image tag in this file after the image is built and pushed, indicating to Flux that a new image is available in ECR.

But before we see that in action, let’s install Flux and its other controllers to your cluster.


* Set up a GitHub token:

In order to create the reconciliation repository for Flux, you'll need [a personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) for your GitHub account that has permissions to create repositories. The token must have all permissions under repo checked off. Copy and keep your token handy and in a safe place.

* On the command line, export your GitHub personal access token and username:
```
export GITHUB_TOKEN=[your-github-token]
export GITHUB_USER=[your-github-username]
```
4. Create the Flux reconciliation repository. In this case you’ll call it fleet-infra, but you can call it anything you want.

In this step, a private repository is created and all of the controllers will also be installed to your EKS cluster. When bootstrapping a repository with Flux, [it’s also possible to apply only a sub-directory](https://toolkit.fluxcd.io/cmd/flux_create_kustomization/) in the repo and therefore connect to multiple clusters or locations on which to apply configuration. To keep things simple, this example sets the name of one cluster as the apply path:
```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=[cluster-name] \
  --personal
```
[Flux version 2](https://www.weave.works/blog/gitops-with-flux-v2) let’s you easily work with multiple clusters and multiple repositories. New cluster configurations and applications can be applied from the same repo by specifying a new path for each.

Once it’s finished bootstrapping, you will see the following:
```
► connecting to github.com
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace …..
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
```

Check the cluster for the flux-system namespace with:

`kubectl get namespaces`

```
NAME              STATUS   AGE
default           Active   5h25m
flux-system       Active   5h13m
kube-node-lease   Active   5h25m
kube-public       Active   5h25m
kube-system       Active   5h25m
```

* Clone and then cd into the newly created private fleet-infra repository:

`git clone https://github.com/$GITHUB_USER/fleet-infra
cd fleet-infra`

* Connect the guestbook-gitops repo to the fleet-infra repo with:
```
flux create source git [guestbook-gitops] \
  --url=https://github.com/[github-user-id/guestbook-gitops] \
  --branch=master \
  --interval=30s \
  --export > ./[cluster-name]/[guestbook-gitops]-source.yaml
```
Where,

  * `[guestbook]` is the name of your app or service
  * `[cluster-name]` is the cluster name
  * `[github-user-id/guestbook-gitops]` is the forked guestbook repository

Configure a Flux kustomization to apply to the `./deploy` directory from your new repo with:
```
flux create kustomization guestbook-gitops \
 --source=guestbook-gitops \
 --path="./deploy" \
 --prune=true \
 --validation=client \
 --interval=1h \
 --export > ./[cluster-name]/guestbook-gitops-sync.yaml
```
Commit all of the changes to the repository with:

`git add -A && git commit -m "add guestbook-gitops deploy" && git push`

`watch flux get kustomizations`

You should now see the latest revisions for the flux toolkit components as well as the guestbook-gitops source pulled and deployed to your cluster:
```
NAME            REVISION                                        SUSPENDED       READY   MESSAGE
flux-system     main/e1c2a084e398b9d36ce7f5067c44178b5cf9a126   False           True    Applied revision: main/e1c2a084e398b9d36ce7f5067c44178b5cf9a126
guestbook       master/35147c43026fec5a49ae31371ae8c046e4d5860e False           True    Applied revision: master/35147c43026fec5a49ae31371ae8c046e4d5860e
```

Check that all of the services of the guestbook are deploying and running in the cluster with:

`kubectl get pods -A`

Where you should see something like the following:
```
NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
default       redis-master-545d695785-cjnks   1/1     Running   0          9m33s
default       redis-slave-84548fdbc-kbnj      1/1     Running   0          9m33s
default       redis-slave-84548fdbc-tqf52     1/1     Running   0          9m33s
flux          flux-75888db95c-9vsp6           1/1     Running   0          17m
flux          memcached-86869f57fd-42f5m      1/1     Running   0          17m
kube-system   aws-node-284tq                  1/1     Running   0          41m
kube-system   aws-node-csvs7                  1/1     Running   0          41m
kube-system   coredns-59dfd6b59f-24qzg        1/1     Running   0          48m
kube-system   coredns-59dfd6b59f-knvbj        1/1     Running   0          48m
kube-system   kube-proxy-25mks                1/1     Running   0          41m
kube-system   kube-proxy-nxf4n                1/1     Running   0          41m
```

### Flux troubleshooting tips

If you have any trouble with configuring Flux, run these commands to see how everything is setup and help you troubleshoot any set up errors:

`flux get sources git`

`flux get kustomizations`

To see all of the commands you can run with Flux run:

`flux --help`

There are many new features in Flux version 2 that you can explore on your own in the Flux version 2 documentation.

4. Make a change to the guestbook app and deploy it with a `git push`

**Note:** If you’d like to see the Guestbook UI before you make a change, kick off a build by adding a space to the index.html file and pushing it to git.

Let’s make a simple change to the buttons on the app and push it to Git:

Open the `index.html` file and change line 15:
```
<button type="button" class="btn btn-primary btn-lg" ng-click="controller.onRedis()">Submit</button>
```
To:
```
<button type="button" class="btn btn-primary btn-lg btn-block" ng-click="controller.onRedis()">Submit</button>
```
Once you’ve made the change to the buttons do a, `git add`, `git commit` and `git push`.   Click on the Actions tab in your repo to watch the pipeline test, build and tag the image.

Now check to see that the new frontend images are deploying:

`kubectl get pods -A`
```
NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
default       frontend-6f9b84d75d-g48hf       1/1     Running   0          95s
default       frontend-6f9b84d75d-ncqj6       1/1     Running   0          84s
default       frontend-6f9b84d75d-v5pfs       1/1     Running   0          107s
default       redis-master-545d695785-r8ckm   1/1     Running   0          58m
default       redis-slave-84548fdbc-nk4mf     1/1     Running   0          58m
default       redis-slave-84548fdbc-vvmws     1/1     Running   0          58m
flux          flux-75888db95c-pnztj           1/1     Running   0          61m
flux          memcached-86869f57fd-hhqnk      1/1     Running   0          61m
kube-system   aws-node-bcw7j                  1/1     Running   0          67m
kube-system   aws-node-gt52t                  1/1     Running   0          67m
kube-system   coredns-6f6c47b49d-57w8q        1/1     Running   0          74m
kube-system   coredns-6f6c47b49d-k2dc5        1/1     Running   0          74m
kube-system   kube-proxy-mgzwv                1/1     Running   0          67m
kube-system   kube-proxy-pxbfk                1/1     Running   0          67m
```
### Display the Guestbook application

Display the Guestbook frontend in your browser by retrieving the URL from the app running in the cluster with:

`kubectl get service frontend`

The response should be similar to this:
```
  NAME       TYPE        CLUSTER-IP      EXTERNAL-IP        PORT(S)        AGE
  frontend   ClusterIP   10.51.242.136   109.197.92.229     80:32372/TCP   1m
```

Now that you have Flux set up, you can keep making changes to the UI, and run the change through GitHub Actions to build and push new images to ECR. Flux will notice the new image and deploy your changes to the cluster, kicking your software development into overdrive.

## Cleaning up

To uninstall flux run:

`brew uninstall flux`

To delete the cluster, run:

`eksctl delete cluster --name [name of your cluster]`

## Final Thoughts

These two posts explain GitOps concepts and its origins. We then demonstrated how to pull together a GitOps pipeline for application deployments. This is one example of how to leverage GitOps.  To see even more use cases and examples of how you can leverage GitOps, see the [Flux version 2 docs](https://toolkit.fluxcd.io/get-started/#staging-workflow).


