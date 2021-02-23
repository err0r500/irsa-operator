# IRSA operator 

![CI](https://github.com/VoodooTeam/irsa-operator/actions/workflows/ci.yml/badge.svg)

A Kubernetes operator to manage IAM roles & policies needed for IRSA, directly from your EKS cluster

This project is built using the Kubernetes [operator SDK](https://sdk.operatorframework.io/)

## Caveat
- oidc must be enabled on your EKS cluster 

## Example

This CRD will allow any pod using the `serviceAccount` named `s3-get-lister` to `Get` and `List` all objects in the s3 bucket with ARN `arn:aws:s3:::test-irsa-4gkut9fl`

```
apiVersion: irsa.voodoo.io/v1alpha1
kind: IamRoleServiceAccount
metadata:
  name: s3-get-lister 
spec:
  policy: 
    statement:
      - resource: "arn:aws:s3:::test-irsa-4gkut9fl"
        action:
          - "s3:Get*"
          - "s3:List*"
```

What this operator does (from a user point of view) :
- create an IAM Policy with the provided statement
- create an IAM Role with this policy attached to it
- create a serviceAccount named as specified with the IAM Role capabilities

you can use the serviceAccount created by the irsa-operator by simply setting its name in your pods `spec.serviceAccountName`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: irsa-test
  name: irsa-test
spec:
  selector:
    matchLabels:
      app: irsa-test
  template:
    metadata:
      labels:
        app: irsa-test
    spec:
      serviceAccountName: s3-get-lister # <- HERE
      containers:
      - image: amazon/aws-cli
        name: aws-cli
        command: ["aws", "s3", "ls", "arn:aws:s3:::test-irsa-4gkut9fl"]
```

## (manual) installation of the operator

### pre-requisites
- kubectl configure to talk to the EKS where you want to install the operator
- a docker registry where the EKS cluster can pull the operator image you'll build
- an IAM role with the ability to create policies, roles, attach policies (use its arn instead of the placeholder `<role_arn>`)

### build the docker image of the controller and push it to an ECR

```
make docker-build docker-push IMG=<image_tag>
```

_NB : it will run all the tests before building the image_

### install with Helm 
```
helm install irsa-operator --set image=<image_tag> --set rolearn=<role_arn> --set oidcProviderARN=<oidcProviderARN> --set clusterName=<desired_identifier> ./config/helm/
```
_NB_ : 
- the rolearn is the role the operator will use, see ./_example/terraform/main.tf for an example of IAM role & policy
- the oidcProviderARN is known at cluster creation if `oidc` is enabled
- the `clusterName` is used to avoid name collisions between AWS IAM resources created by different EKS running in the same account, you can use whatever value you want (most likely the EKS cluster name)

#### check

you can access operator's logs there :
```
k logs deploy/irsa-operator-controller-manager -n irsa-operator-system -c manager -f
```

### deploy a resource that uses the iamroleserviceaccount CRD 

```
helm install s3lister --set s3BucketName=<bucket_name> ./_example/k8s
```

#### check 
you can access logs of your pod

```
kubectl logs --selector=app=s3lister
```

if you see the listing of your s3 `<bucket_name>`, congratulations ! the pod has been able to achieve this using the abilities you gave it in your `IamRoleServiceAccount.Spec` !

## project structure
this project follows the `operator SDK` structure : 
- CR types are declared in `./api/<version>`, the `zz_generated...` file is autogenerated based on other CRs using the `make` command
- Controllers (handling reconciliation loops) are in `./controllers/`, one controller per CR.

## architecture

Here's how IRSA works and how the irsa-operator interfaces with it

![](./_doc/architecture-diagram.png)

## model

the way this operator works is described [there](./_doc/model/IrsaOperator.pdf)

## work on the project

### resources
- [kubebuilder](https://book.kubebuilder.io/)
- [kubernetes operator concurrency model](https://openkruise.io/en-us/blog/blog2.html)

### tests
- check test coverage in your browser with `go tool cover -html=cover.out`

## Release process
### Publish docker image
- create a release with the name `v<version>`
- it will trigger the `publish-docker` workflow and push the docker image to github artefacts

### Publish the helm chart
if the previous step went fine
- update [./config/helm/irsa/Chart.yaml](./config/helm/irsa/Chart.yaml) and set the version to <version>
- it will trigger the `chart-release` workflow, publish the helm chart and create a release called `helm-v<version>`






