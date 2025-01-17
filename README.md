# Application Deployment Proposal in the EKS Cluster with GitOps

## 1. Initial Assumptions

During the design phase of the solution, the following assumptions were made:

- Resources are deployed in the **EKS cluster** on AWS in **eu-central-1** region.
- The **Ingress Controller** used in the cluster is the **AWS Load Balancer Controller**.
- At the time of the application's deployment, the cluster has the **External Secret Operator** deployed and operational.
- **ArgoCD** and all its core components are correctly deployed and configured in the cluster.

---

## 2. Content of this repository

This repository has two directories:

- **helm** - this is the directory where a **tidio-app-chart** can be found. This is the main helm chart that has the complete set of manifests for deploying the solution
- **additional_resources** - in this directory, raw manifest files can be found. The mian purpose is to have already generated manifests somewhere, just in case.

## 3. Deployment of the Configuration to the Cluster

### a. External resources creation

Before deploying the application, the first step is to create an **SSL certificate** upfront, to be able to set the secure communication between the **user** and the **application**. 

For this purpose, we should be able to use a dedicated IaC repository with infrastructure code (e.g., Terraform) and configured CI/CD processes. In such a repository, after selecting the appropriate location in the directory structure, we add a configuration responsible for creating the certificate in AWS Secrets Manager.

---

### b. Application Deployment

When deploying our application to the cluster we should follow the **GitOps approach**. This implies that the state of resources defined in the repository should match the resources created in the cluster.
The tool used for GitOps in this case is **ArgoCD**, which will manage the deployment and ensure the desired state of resources is maintained.
To simplify the application deployment process as much as possible, application manifests have been packaged as a **Helm Chart**.

#### 1. Build Helm Chart and sent it to the registry

As the first step, application configuration should be properly build and send to the registry. Since Helm 3 supports OCI registires it is possible to keep it in eg. AWS ECR. An appropriate approach would be to create a dedicated repository for the Helm Charts code, allowing it to be independently developed and tagged along with the release of new versions. We should have CI/CD process configured with the repository, it would be responsible for syntax validation, configuration correctness, building, and pushing the Helm Chart to the registry.

To manually prepare the Helm Chart and push it to, for example, ECR, we can use the commands below:
```
helm package tidio-app-chart

Output: Successfully packaged chart and saved it to: /Users/username/tidio-app-chart-0.1.0.tgz

helm push tidio-app-chart-0.1.0.tgz oci://aws_account_id.dkr.ecr.region.amazonaws.com/
```
#### 2. (Optional) Configure the App of Apps pattern in ArgoCD.

The App of Apps pattern is used to monitor a specific location in the repository and automatically create new applications based on new **ArgoCD Application** objects. Such a location is typically a main GitOps repo for the cluster.

If applications are not yet created in this way, a detailed configuration description can be found [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

#### 3. Create an ArgoCD Application object configuration and use the previously built Helm Chart in it.

To deploy the configuration using ArgoCD, it is necessary to create an additional manifest for the Application object, which will be responsible for deploying the Helm Chart. If the App of Apps is properly configured in ArgoCD, this should be the only task to perform. After placing the correct Application object configuration in the configured location monitored by App of Apps, the entire solution should be successfully deployed in the cluster.

The example manifest for the Application resource is presented below:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tidio-app
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: tidio
  source:
    repoURL: 111222333444.dkr.ecr.eu-central-1.amazonaws.com
    targetRevision: 0.1.0
    chart: tidio-app-chart/tidio-app-chart
    helm:
      releaseName: tidio-app
      version: v1
      valueFiles:
      - values-production.yaml
  destination:
    namespace: kube-system
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```



---

## Key Tools and Services

- **AWS EKS**: Managed Kubernetes service for resource orchestration.
- **AWS Load Balancer Controller**: Ingress Controller for routing traffic.
- **AWS Secrets Manager**: Secure storage for SSL certificates and secrets.
- **External Secret Operator**: Synchronizes secrets from AWS Secrets Manager to Kubernetes.
- **ArgoCD**: GitOps tool to manage Kubernetes resources and deployments.
- **Helm**: For packaging and templating Kubernetes Manifests

---

## Summary

To sum up, when using the GitOps approach and with the heavy lifting handled by tools like ArgoCD, there is no need for many commands or manual operations to deploy a prepared solution in the cluster. This also brings additional benefits, such as the ability to easily roll back a specific release or a very simple process for releasing new versions of the application.
