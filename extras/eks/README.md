# Bank of Anthos — AWS EKS Deployment

This directory contains a [Kustomize](https://kustomize.io/) overlay that adapts the
base `kubernetes-manifests/` for deployment on **Amazon Elastic Kubernetes Service (EKS)**.

## What this overlay changes

| Change | Reason |
|--------|--------|
| Removes `iam.gke.io/gcp-service-account` annotation from `ServiceAccount` | GKE Workload Identity is not applicable on EKS |
| Sets `ENABLE_TRACING=false` on all services | Disables export to Google Cloud Trace, which requires GCP credentials |
| Sets `ENABLE_METRICS=false` on applicable services | Disables export to Google Cloud Monitoring, which requires GCP credentials |

## Prerequisites

- An EKS cluster (1.24+) with `kubectl` configured to access it.
  Use [eksctl](https://eksctl.io/) or the [AWS Console](https://console.aws.amazon.com/eks/) to create one.
- [kubectl](https://kubernetes.io/docs/tasks/tools/) v1.24+
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) v5+ (or `kubectl` v1.14+ which bundles kustomize)

## Quickstart

1. **Clone the repository.**

   ```sh
   git clone https://github.com/GoogleCloudPlatform/bank-of-anthos
   cd bank-of-anthos/
   ```

2. **Configure `kubectl` to target your EKS cluster.**

   ```sh
   aws eks update-kubeconfig --region <REGION> --name <CLUSTER_NAME>
   ```

3. **Apply the JWT secret.**

   ```sh
   kubectl apply -f ./extras/jwt/jwt-secret.yaml
   ```

4. **Deploy using the EKS kustomize overlay.**

   ```sh
   kubectl apply -k ./extras/eks
   ```

5. **Wait for the pods to be ready.**

   ```sh
   kubectl get pods
   ```

   After a few minutes, all pods should reach `Running` status.

6. **Get the frontend URL.**

   ```sh
   kubectl get service frontend | awk '{print $4}'
   ```

   Visit `http://<EXTERNAL-IP>` in a browser to access Bank of Anthos.
   AWS provisions an Elastic Load Balancer for the `frontend` service; DNS
   propagation may take a minute or two.

## Optional: Use AWS Network Load Balancer (NLB)

By default, `type: LoadBalancer` on EKS creates a Classic Load Balancer.
To use a Network Load Balancer instead, annotate the `frontend` service after deployment:

```sh
kubectl annotate service frontend \
  service.beta.kubernetes.io/aws-load-balancer-type=nlb
```

Or install the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
and add the annotation `service.beta.kubernetes.io/aws-load-balancer-type: "external"` via an
additional kustomize patch.

## Optional: Enable AWS IAM Roles for Service Accounts (IRSA)

If you need the Bank of Anthos pods to call AWS services (e.g., Amazon RDS,
Secrets Manager), you can replace the GKE Workload Identity annotation with the
equivalent EKS annotation:

```sh
kubectl annotate serviceaccount bank-of-anthos \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

## Teardown

```sh
kubectl delete -k ./extras/eks
kubectl delete -f ./extras/jwt/jwt-secret.yaml
```
