# Tech Chapter Log / Guide
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

## Important folders and files


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
```monkey:=
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
```monkey:=
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
```monkey:=
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
```monkey:=
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
```monkey:=
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

```m=
kubectl create namespace argo
```

```m=
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

# Week 5
