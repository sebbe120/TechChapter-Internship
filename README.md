# Tech Chapter Log / Guide
This document details our experiences during the internship, and includes both technical notes and also in general the things we are working on. It describes some of the problems we encountered and how we tried to fix them.

[TOC]

# Week 1 Kubernetes in AWS & signed commits

## Manual Setup and things we learned
We are using an EKS Cluster in AWS.
install aws cli, kubectl and eksctl
run ```aws configure sso``` to get authentication

edit file  ~/.aws/config (default location) so that you have a default profile
after this, you can make AWS create / edit kubeconfig by running ```aws eks update-kubeconfig --region region-code --name my-cluster```

aws-auth ConfigMap user access:
https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html


It may be required to use the ```--profile {user}``` flag in case the default profile does not have access to AWS.

It may be required to use the ```-n {namespace}``` flag in case the "objects" you are trying to interact with are in a different namespace.

## Cluster
There are two ways to deploy a cluster: using the AWS Management Console (GUI) or the commandline.
As described above the AWS Management Console requires the user to setup many configurations like the ConfigMap/aws-auth for access.

Using the cli makes it easier, since it creates most of the configurations for you. CLI Tool: eksctl.

Creating a cluster:
```eksctl create cluster --name {my-cluster} --region {region-code}```

This also updates the kube config file to include the new cluster for you.
But if you still need to change the kube config:
```aws eks update-kubeconfig --region {region-code} --name {my-cluster}```

Check if the cluster is working:
```kubectl get svc```

## Deployment
Guide:
https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html

If you need a new namespace:
```kubectl create namespace eks-sample-app```

Then you need to create a deployment file like this example:
```yaml:=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-linux-deployment
  namespace: eks-sample-app
  labels:
    app: sample-linux-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-linux-app
  template:
    metadata:
      labels:
        app: sample-linux-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
                - arm64
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:1.23
        ports:
        - name: http
          containerPort: 80
        imagePullPolicy: IfNotPresent
      nodeSelector:
        kubernetes.io/os: linux
```
And apply it using:
```kubectl apply -f {path_to_file}\{filename}.yaml```

Create a service file like this:
```yaml:=
apiVersion: v1
kind: Service
metadata:
  name: eks-sample-linux-service
  namespace: eks-sample-app
  labels:
    app: eks-sample-linux-app
spec:
  selector:
    app: eks-sample-linux-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

And apply it using:
```kubectl apply -f {path_to_file}\{filename}.yaml```

View all ressources to see if it is working:
```kubectl get all -n {namespace}```


## Loadbalancer
```yaml:=
apiVersion: v1
kind: Service
metadata:
  name: mr-test-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: mr-test-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
## Nodeport
```yaml:=
apiVersion: v1
kind: Service
metadata:
  name: eks-sample-linux-service-nodeport
spec:
  type: NodePort
  selector:
    app: eks-sample-linux-deployment
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```


## Ingress
```yaml:=
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: service-2048
              port:
                number: 80
```


## Useful Commands
### Kubernetes

```ctrl + d```

```kubectl create namespace {namespace}```

```kubectl -n {namespace} describe pod {pod_identifier}```

```kubectl exec -it {pod_identifier} -n {namespace} -- /bin/bash```

```curl eks-sample-linux-service```

```kubectl apply -f {path_to_file}```

```kubectl get all -n {namespace}```
```kubectl -n {namespace} describe service {servicename}```

### AWS CLI
```aws sts get-caller-identity```
```aws eks update-kubeconfig --region region-code --name my-cluster```
Antag en anden rolle:
```aws sts assume-role --role-arn arn:aws:iam::851725286774:role/praktikanterne_der_endnu_ikke_ved_hvad_de_laver --role-session-name praktikant_2 --profile Contributor-851725286774```

## If stuff goes wrong
Try appending -n {namespace} after your command :-1: 

Error with ```aws sts get-caller-identity```:
"An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation: The security token included in the request is invalid."

Solution: remember to run```set aws_session_token = {token}```

## Git ssh access and signed commits
Det krævede lidt konfiguration af git, at arbejde for Techchapter
Blandt andet krævede det signed commits, hvilket var nyt for os begge.
Nedenfor ses et eksempel på konfiguration af git:
```
git config --global user.name username
git config --global user.email email
git config --global user.signingkey /path/to/signingkey.pub
git config --global gpg.format ssh
git config --global commit.gpgsign true
```

## SSH access to git through CLI
To use SSH acccess to github:
``` git clone git@github.com:techchapter/hypervisor-herd.git```
instead of the "usual" method:
```git clone https://github.com/techchapter/hypervisor-herd```

Screenshot is a reminder on where to find the correct format.
![image](https://hackmd.io/_uploads/By5iDtGYp.png)


# Week 2 Kubernetes and Terraform

We use Docker Desktop with Kubernetes enabled.

Argo Workflows Guide:
https://argo-workflows.readthedocs.io/en/latest/quick-start/

Download the exe and add to Windows Path:
https://github.com/argoproj/argo-workflows/releases/tag/v3.5.2

```
kubectl create namespace argo
```

```!
kubectl patch deployment argo-server --namespace argo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": ["server", "--auth-mode=server"]}]
```

We quickly changed this approach, to get the configmap configured for cluster access (we were first succesfull at the end of week 3).

## Signed commits github

generate a keypair (using eg. ssh keygen, pgp, or whatever.. )
Then tell the git client:

- The type of certificate you are using
- The location of the certificate
- possibly also if the global default is to sign all commits

https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key


What worked (Sebastian):
- Do not use GPG as it is a scam
- use the ssh key we generated on day one
- set format to ssh
- signingkey = C:\\Users\\sebas\\.ssh\\{someid}.pub
- Remove passwords - ie. ssh-keygen -p and press Enter when a new password is promted, to remove password
    - This is because GitHub Desktop does not support password keys
- Use GitHub Desktop to push
- Or just use git bash since that also works

What worked (Michael):
Everything, just remember to set options in git config!
  - user.name
  - user.email
  - auth method
  - path to cert


## <span style="color:blue">Terraform</span>
Remember to use the "**terraform fmt**" command to make sure that the format is the same as the other persons - to make sure that github does not recognize changes that aren't really changes

Terraform flow:
- Write Terraform code
- Run terraform fmt
- Run terraform init
- Run Terraform Plan

Only run "terraform apply" when you are ready to deploy

Things done:
Getting started with Terraform "Pluralsight" certificate videos
"Refactorized" the pippiio module with Joachim.
Started a new Terraform projekt with a basic cluster

### config-map for aws_auth
It still does not work.

Changed the whitelist to allow all ips, since only the Techchapter local ip was allowed (hardcoded), and Terraform was not.


# Week 3 and Week 4

We got permissions to run ```terraform destroy``` in the cloud, which massively speeds up taking down the cluster and creating a new one.

We tried to create a (OICD (needed for terraform apply locally)) policy for our contributor role, but it is protected by AWS. We then tried to create the same policy for our custom role.

We also copied all of the IAm policies from the contributor role into the custom role.

We created a shell script that gets credentials after assuming our intern role. It then sets these as environmetal variables and updates the .kube/config.

Michael got access but Sebastian did not, which indicates that there still might be something wrong with the configmap.

We now changed the approach to avoid using tfcloud. Instead we use terraform locally, and save the state file in an s3 bucket on amazon.

We got this to work with assuming the praktikant role (for Michael). After this we thought that we had configured it correctly, but deploying to tfcloud resulted in no succes.

We finally got a cluster with a working configmap after switching away from the vpc/pippiio module, and using the "default" terraform eks and vpc modules. This meant that we could now access the cluster with kubectl with any role within the configmap.

After this we created a github project with a kanban board for managing tasks.

## Subnetting
As the original modules appears to be outdated, we've been tasked to use other modules (official modules)
These require some calculation of subnetes, CIDR ranges and such.
This was a task we were ill prepared for.

If we set CIDR to 192.168.0.0/20 and subnets to 192.168.0.0/24, that means we will have 4 bits available for allocating subnets (16 possible subnets) and (32-24) 8 bits available to allocate addresses inside each subnet (256 addresses).


## CoreDNS Error
If you do not have at least 2 nodes in your cluster, cordns will give the error:
"Error: unexpected EKS Add-On (pippi-cluster:coredns) state returned during creation: timeout while waiting for state to become 'ACTIVE' (last state: 'DEGRADED', timeout: 20m0s) [WARNING] Running terraform apply again will remove the kubernetes add-on and attempt to create it again effectively purging previous add-on configuration"


## Helm provider
The plan was to get used to Helm so we could use it to deploy Helm Charts into the cluster. We got the basic nginx static page to work and could access using a url.

We started looking at how helm charts are created, but this seemed like a big task, and was also not necessarily within the scope of the current task - Argo Workflows.


## Argo Workflows

We could not deploy Argo with terraform, but we could deploy it using the helm cli tool. We could however not get access to the cluster, which might be because of some ingress shenenasdnfas.


Installing via Helm CLI:
1. `helm repo add bitnami https://charts.bitnami.com/bitnami`
2. `helm repo update`
3. `helm install bitnami/bitnami/argo-workflows --generate-name`

Continues in week 5

# Week 5
## Argo Workflows continued

Portforward:
`kubectl port-forward --namespace default svc/argo-wf-argo-workflows-server 8080:80 & echo "Argo Workflows UI URL: http://127.0.0.1:8080/"`

This however results in an error:
```!
Forwarding from 127.0.0.1:8080 -> 2746
Forwarding from [::1]:8080 -> 2746
Handling connection for 8080
E0129 12:11:24.456670   18812 portforward.go:409] an error occurred forwarding 8080 -> 2746: error forwarding port 2746 to pod bc8f38250fa892fa902cb2f30081e02e332e02e92285c596ad7e79808f8bd543, uid : failed to execute portforward in network namespace "/var/run/netns/cni-235c801b-301d-1f7e-6d22-5428662d976e": failed to connect to localhost:2746 inside namespace "bc8f38250fa892fa902cb2f30081e02e332e02e92285c596ad7e79808f8bd543", IPv4: dial tcp4 127.0.0.1:2746: connect: connection refused IPv6 dial tcp6: address localhost: no suitable address found
error: lost connection to pod
```
which we think? might be caused by kubectl using the wrong port when forwarding???
This was not the case, since we found out the problem was with the argo postgres pod. The 2 argo pods depends on the argo postgres pod, so they wont work until the postgres pod is working.
The problem with the postgres pod was that it would not start up since it was missing a volume to bind to. The automatic bind would not work, since the driver (aws-ebs-csi-driver ) for programatically creating / binding volumes in aws is disabled by default. We added it to the cluster_addons, and now the postgres pod started.

```!
# Needed by the aws-ebs-csi-driver
iam_role_additional_policies = {
  AmazonEBSCSIDriverPolicy = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}
```

After the postgres pod was running, the 2 argo pods started succesfully after a reboot.
Now the portforwarding worked, and we could go to the argo workflows homepage.

We were not authorized to do anything so we started looking at Ingress / alb Loadbalancing.

## Ingress
Deployment of k8s ingress failed due to this error message:
```!
Warning  FailedBuildModel  12m  ingress  Failed build model due to WebIdentityErr: failed to retrieve credentials
caused by: ValidationError: Request ARN is invalid
```

Hoping for a solution from this [aws article](https://repost.aws/knowledge-center/eks-load-balancer-webidentityerr) that says it's related to service accounts

These links fixed this error:
https://stackoverflow.com/questions/66405794/not-authorized-to-perform-stsassumerolewithwebidentity-403
https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html

We changed the trust policy for the praktikant role to include "sts:AssumeRoleWithWebIdentity", and it fixed this problem.
```json!
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::851725286774:role/aws-reserved/sso.amazonaws.com/eu-north-1/AWSReservedSSO_Contributor_14bf02704449cc6b",
                "Service": "eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::851725286774:role/aws-reserved/sso.amazonaws.com/eu-north-1/AWSReservedSSO_Contributor_14bf02704449cc6b",
                "Service": "eks.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.eu-north-1.amazonaws.com/id/4311742964249D15EB5FBF28832B42C4:sub": "system:serviceaccount:default:argo-wf-argo-workflows-server"
                }
            }
        }
    ]
}
```

We came to the conclusion that we had no idea what to do, and even if the ingress controller was deployed correctly.

We started following this guide:
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

Step-by-step:
```!
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
```!
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```
Be careful about using override as it overwrites existing roles
```!
eksctl create iamserviceaccount --cluster=argo-wf-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::851725286774:policy/AWSLoadBalancerControllerIAMPolicy --approve --override-existing-serviceaccounts
```

```
helm repo add eks https://aws.github.io/eks-charts
```

```
helm repo update eks
```

```!
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=argo-wf-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

To check if the load-balancers are working
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

It worked and outputtet an "official" url which we could use to access the argo workflow home page. (we also created a serviceaccount for ingress but we don't know if it had any impact).
We still need to set up authentication for it.

We discovered how to deploy helm charts with custom values. By using the command `helm upgrade -f values.yaml argo-wf argo-workflows-6.3.1.tar.gz` we can inject custom values into the chart before deploying. Remember to restart the pods afterwards to see the changes.

**If deleting ingress does not work, check if a finalizer exists as they might stop the deletion process.**

Next steps:
* Use helm provider to deploy load balancer via terraform
* Use helm provider to deploy ingress.yaml via terraform
* Find out how to deploy argo-workflow with terraform
* * Find out how to get argo-workflow setup with sso login
* Deploy to terraform cloud and create a new releas v0.2alpha

Getting the load balancer to work in terraform proved to be a difficult task.

We were succesfull in creating the necessary service account, but getting the load balancer to work did not happen - We also added service account creation for ALB in the terraform code.
We started by usinf the "aws_lb" which created a load balancer, but found out after a lot of testing, that it only created a blank load balancer, with nothing configured at all.
This "module" could definitely work, but required much knowledge about how to create a load balancer from scratch where EVERYTHING need to be manually created.

We finally got it to work with:
```=
resource "helm_release" "load_balancer_controller" {
  name       = "tf"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace = "kube-system"

  set {
    name = "clusterName"
    value="argo-wf-cluster"
  }
  set {
    name = "serviceAccount.create"
    value="false"
  }
  set {
    name = "serviceAccount.name"
    value="tf-aws-load-balancer-controller"
  }
}
```
since the only thing missing was to actually "bind" the service account the the load balancer. This module also created everything for us so we did not need to configure much.

We then got ingress to work in terraform, by using the "kubernetes_manifest" ressource, and using yamldecode(ingress.yaml):
```hcl=
resource "kubernetes_manifest" "ingress-manifest" {
  manifest   = yamldecode(file("../ingress.yaml"))
  depends_on = [kubernetes_namespace.argo]
}
```

We discovered that the "kubernetes_manifest" resource contacts the kubernetes api when it is trying to create. This however will never work on a clean cluster install, as terraform does not know that that dependency exists, so it fails.
We tried "kubectl_manifest" instead: https://registry.terraform.io/providers/gavinbunney/kubectl/latest/docs
And it worked, and is also tracked in terraform.

Now we just need to get argo workflows to work, and we can try to deploy in tfcloud and create a new release: v0.2alpha.

We now were facing a problem with terraform destroy. Because of some dependencies that terraform does not know about, it cannot delete an object that other objects depend on:
* The VPC cannot be destroyed because it depends on the subnets
* The subnets connot be destroyed because they depend on the ALB
* The ALB cannot be destroyed since terraform does not know it exists

We think that because we deploy the ALB with a helm chart in terraform, it creates some ressources that terraform does not track in the state. So when we run terraform destroy, it will fail.

<span style="color:red">**[Short term fix]**</span>
What to do to make terraform destroy work:
* Manually destroy the load balancer in aws
* Manually destroy the subnets in aws
* Manually destroy the security groups

# Week 6 

## <span style="color:darkblue">**Ikea**</span>
We started the week by assembling Ikea furniture. We assembled noice cancelleres which were to be placed on the tables, as the room we work in has bad accoustics. The product was the eilif:
https://www.ikea.com/dk/da/p/eilif-skaerm-til-skrivebord-gra-00466937/
We were assigned this role since we were waiting for other tasks to be ready. It was a role which we knew we had the right experience to fulfill, so we were happy to oblige. It did not take long deploying the noice cancellers to the tables, but we soon realized that there were ordered too few of them. This was not a critical problem, but it was a setback nonetheless. We tried using root cause analysis, to figure out what happened and where it went wrong but to no avail. We held a meeting afterwards to discuss the current state of the situation and we came to the conclusion that there needed to be ordered some more noice cancellers so all the tables could be covered. This is especially important as each person is equal, so the people who are missing the noice cancellers, might feel left out - which is definitely not a good situation to be in. Be it for the victim or the workplace. We don't want to create a hostile working environment as that can have fatal effects on the company.


## Problem with "corrupted" namespace
We tried destroying the cluster, but ran into new problems. Now the argo namespace was stuck terminating endlessly and caused the terraform destroy to fail.

After deploying the cluster again, we ran into a new problem we never have seen before.
For some reason the argo pods could not start at all, and output an error with postgres user. We don't know what causes this.
Somehow deploying argo in the default namespace, fixed this error, and we can now try to progress again.
Disabling auth works.
Deploying argo in a namespace that is not the "argo" namespace works.

## Argo
We found out how to login to argo workflows. We set the auth mode to client which takes a kubernetes bearer token as input. We followed this guide:
https://argo-workflows.readthedocs.io/en/latest/access-token/

We could now login but did not have access to creating workflows and such. After a couple of hours we found out that a magic string with the value "argo" was actually the namespace the service account would "work" in. We did not notice and assumed it was to specify the argo deployment.
After changing this to the actual namespace it worked.

**Full step by step (with the name argo-client)**
argo-client-role.yaml:
```yaml=
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: workflows
  name: argo-client
rules:
- apiGroups:
    - argoproj.io
  resources:
      - pods
      - workflows
      - workflows/finalizers
      - cronworkflows
      - cronworkflows/finalizers
      - workfloweventbindings
      - workfloweventbindings/finalizers
      - workflowtaskresults
      - workflowtaskresults/finalizers
      - workflowtasksets
      - workflowtasksets/finalizers
      - workflowtasksets/status
      - workflowtemplates
      - workflowtemplates/finalizers  
      - eventsources
      - sensors
  verbs: ["get", "watch", "list"]
```
Cli commands to deploy:
```yaml!
~~kubectl create role argo-client --verb=list,update --resource=workflows.argoproj.io -n workflows~~

kubectl create sa argo-client -n workflows

kubectl create rolebinding argo-client --role=argo-client --serviceaccount=workflows:argo-client -n workflows
```
```yaml!
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  namespace: workflows
  name: argo-client.service-account-token
  annotations:
    kubernetes.io/service-account.name: argo-client
type: kubernetes.io/service-account-token
EOF
```
```!
ARGO_TOKEN="Bearer $(kubectl get -n workflows secret argo-client.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"

echo $ARGO_TOKEN
```

## Moving back to tfcloud

As usual we had problems moving from s3 bucket to the tfcloud backend. We think it is bacause tfcloud still has a state. It tries to apply out changes but says there are drifts with many resources. It ultimately fails when it tries to contact the kubernetes cluster and cannot reach it - which it can't since it does not exist.

We do not have permissions to operations like destroy so we were kind of deadlocked - until the admin helped us.

We destroyed the cluster and deployed it to tfcloud. Most of the deployment was fine but we got the same issue when it came to the helm release:
```!
Error: Kubernetes cluster unreachable: Get "https://9D78EC50BBB1275595721A6EF091F4DA.gr7.eu-north-1.eks.amazonaws.com/version": getting credentials: exec: executable aws failed with exit code 255
with helm_release.argo_wf
on helm.tf line 13, in resource "helm_release" "argo_wf":
resource "helm_release" "argo_wf" {
```
We discovered that the problem was with the helm provider which looked like this:
```!
provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.cluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      args        = ["eks", "get-token", "--cluster-name", data.aws_eks_cluster.cluster.name]
      command     = "aws"
    }
  }
}
```

We assumed that since deployment worked with an s3 backend, it would also work in tfcloud, but the problem was with the exec statement in the helm provider.
It works fine when deploying with the cli since we have the aws-cli installed, but tfcloud does not, so it cannot run the command and will not get credentials to deploy helm releases.

After changing it to the same credentials as the kubernets provider it worked.
Now the only thing that fails in a full tfcloud deployment is the kubectl_manifest that deploys the ingress for argo.

This was because we did not explicitly create the kubectl provider, so it used default values which was using local resources - which is why it only worked on the s3 bucket.
The provider was identical to the helm provider with credentials, and we added "a "load_config_file = false", and it worked.

We now have a fully working cluster that is deployed only using terraform with argo workflows.

The only missing thing is making the service account and getting the kubernetes bearer token for the argo login, but we are doing that manually for now.

## Argo Workflows

We started by deploying the standard hello world image from docker.
We got a basic workflow for posting slack messages working. We used a slack template, and made a workflow that calls the template.

We discovered how to make workflows eventbased by creating a WorkflowEventBinding which exposes a url to access (fx with curl). This means that we can now start up workflows using http request so that we do not have to submit them with the cli.

We made it work and are now able to send messages to our slack test server.
Just remember to setup webhook and add the bot and such.

We planned taking the next week off from work (we might work from home some of the time), so we did not have a lot of time before the break, so we started researching some other topics.

Michael started looking at how you can deploy workflows with python, and Sebastian tried to create a pod that can run kubectl commands, to refresh bearer tokens.

Kubernetes bearer token yaml file
* Update the patch line to work
* Refresh / revoke the token and make a new one
* Make a cronjob that does the same instead of a deployment

This idea was put on the shelf as there must be a better way to do this, and that it isn't really important now, since we can just use hardcoded values (even though it should be changed in a real product).

# Week 7
We did not do much this week as we took a break and only worked a couple hours here and there.
We started looking at creating github actions. We wanted to create a simple action that triggered on pull requests, and then called the argo api which then started the slack notify workflow to send a message to slack about who created the pull request.

```!=
name: slack-notify-pull-pull-request
run-name: ${{ github.actor }} created a pull request

on: [pull_request]
jobs:
  check-bats-version:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl --request POST \
          --header 'Authorization: ${{ secrets.argotoken }}' \
          --header 'X-Argo-E2E: true' \
          --header 'Content-Type: application/json' \
          --url ${{ secrets.argourl }} \
          --data '{"message": "${{ github.actor }} made a pull-request, please can anyone review?", "channel": "hh-more-spam-please"}'

```

# Week 8 argo workflows

We started looking at how to create workflows that have multiple steps to complete. An example of this is the below workflow that sends a message to a channel in both steps:
```yaml=
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: send-to-multiple-slack-channels
spec:
  ttlStrategy:
    secondsAfterSuccess: 10
  entrypoint: send-messages
  arguments:
    parameters:
    - name: outer-message
      value: Message to send
  templates:
  - name: send-messages
    steps:
    - - name: template-run1
        templateRef:
          name: slack
          template: notify
        arguments:
          parameters:
            - name: message
              value: "{{workflow.parameters.outer-message}}"
            - name: channel
              value: "hh-even-more-spam"
    - - name: template-run2
        templateRef:
          name: slack
          template: notify
        arguments:
          parameters:
            - name: message
              value: "{{workflow.parameters.outer-message}}"
            - name: channel
              value: "hh-more-spam-please"
```

## Automating deployment of workflow templates using github actions
Next step was to try to figure out how to deploy workflows without doing it manually.
The idea was to have a workflows folder in the github repo, and then when a workflow file was added / modified / deleted, it would trigger a github action that would deploy / remove the workflow from argo workflows.
