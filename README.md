<div align="center">

# AutoScaling EKS via Karpenter 

</div>

# Overview
In this repository, we deploy a GPT2 model on EKS. The application is exposed via an ALB (Ingress). Horizontal Pod Scaling is achieved as as well as Node Scaling via Karpenter. User's load testing has been done via Locust.

![](./architecture.svg)

# ECR

We push our images for the model server to (public) ECR repository. 

# Install Tools

## eksctl
```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```

## kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh --version v3.12.3
```

## EKS Node Viewer

```bash
https://github.com/awslabs/eks-node-viewer/releases/download/v0.4.3/eks-node-viewer_Linux_x86_64
chmod 777 eks-node-viewer_Linux_x86_64
```


# Setup Karpenter Autoscaler and Create Cluster


```bash
export REGION=...
export ACCOUNT_ID=...
export CLUSTER_NAME=emlo-s18-cluster
export KARPENTER_VERSION=v0.31.0
export AWS_PARTITION="aws"
```


```bash
curl -fsSL https://raw.githubusercontent.com/aws/karpenter/"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $(mktemp) \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

## Create Cluster

```bash
envsubst < 01_cluster.yaml | eksctl create cluster -f -
```


Now the command `kubectl get all` should give an output.

```bash
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

# Install Karpenter

```bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```


# Create Provisioner

```bash
envsubst < 02_karpenter_prov.yaml | kubectl apply -f -
```

# Metrics Server

```bash
# install metrics server - needed for scaling etc.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```


# Deploy Apps

Key points:
- Service types must be NodePort
- Specify ingress annotations
- Remove host in ingress



```bash
helm install emlo-s18 emlo_s18_helm
```

Note that our deployment also contains the horizontal pod scaler.  
To ensure the service is working, you can port-forward

```bash
kubectl port-forward svc/model-serve 8080:9000
curl localhost:8080
```

# Setup LoadBalancer

## OIDC

enable OIDC on your EKS Cluster
```bash
eksctl utils associate-iam-oidc-provider --region ${REGION} --cluster ${CLUSTER_NAME} --approve

# attach policy
# from https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://03_iam-policy.json
```

## IRSA

```bash
eksctl create iamserviceaccount \
--cluster=${CLUSTER_NAME} \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region ${REGION} \
--approve
```

## Deploy AWS LB using HELM

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=${CLUSTER_NAME} --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

> If the ALB is not working as expected, see the logs of the LB web service via `kubectl logs svc/aws-load-balancer-webhook-service -n kube-system` . Note the ENI that is generating the error. In AWS Console, add the following tag :   
> `kubernetes.io/cluster/emlo-s18-cluster:owned`



# Load Testing

For load testing, run

```bash
locust -f locust.py
```
# Logs

[kubectl get all -A  -o yaml](./logs/q1.md)

[kubectl top pod (before load)](./logs/q_top_no_load.md)

[kubectl top pod (after load)](./logs/q_top_load.md)

[kubectl describe ingress/web-ingress](./logs/q_desc_ing.md)


# AutoScaling Demo

