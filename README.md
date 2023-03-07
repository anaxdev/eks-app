# Create an EKS cluster and Deploy some applications

## Setup the development environment

### Install or update the AWS CLI

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
source ~/.bashrc
aws --version
```

### AWS CLI auto-completion

```sh
which aws_completer
export PATH=/usr/local/bin:$PATH
#source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws
source ~/.bashrc
```

### Install kubectl

```sh
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.22.6/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# kubectl auto-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

kubectl version --short --client
```

### Install other utilities

- Install jq gettext bash-completion moreutils

- Install k9s (UI toool for kubernetes): Refer to https://github.com/derailed/k9s

```sh
K9S_VERSION=v0.26.7
curl -sL https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_x86_64.tar.gz | sudo tar xfz - -C /usr/local/bin
k9s
```

### Install eksctl

```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# eksctl auto-completion - bash
. <(eksctl completion bash)
echo "source <(eksctl completion bash)" >> ~/.bashrc

eksctl version
```


## Create an EKS cluster

### Create a VPC using cloudformation file

```sh
aws cloudformation deploy \
  --stack-name "eks-vpc" \
  --template-file "EKSVPC3AZ.yml" \
  --capabilities CAPABILITY_NAMED_IAM 
```

### Export the environment variables for `eksctl`

```sh
./eksctl_envs_export.sh
```
After running the script above, all the exported environment variables are stored in `~/.bash_profile` file.

### Generate an yaml file for `eksctl`

```sh
./eksctl_yaml_generation.sh
```

This script generates an yaml file named `eks-cluster.yaml`, which is used for creating an EKS cluster using `eksctl` tool.

### Create an EKS cluster

Validate the yaml file

```sh
eksctl create cluster --config-file=eks-cluster.yaml --dry-run
```

Create an EKS cluster using `eksctl`

```sh
eksctl create cluster --config-file=eks-cluster.yaml
```

### Other commands

To re-write the kube config file, run:

```sh
eksctl utils write-kubeconfig --cluster=<cluster-name>
```

To delete the ekc cluster, run:

```sh
eksctl delete cluster --name <cluster-name>
```

### Deploy apps on EKS cluster

#### Add an Ingress Controller (Load Balancer Controller)

- Create a policy

```sh
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

- Create a ServiceAccount with the policy above

```sh
eksctl create iamserviceaccount \
    --cluster eks-demo \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```

- Install cert-manager

```sh
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```

- Download the load balancer controller yaml file

```sh
wget https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_full.yaml
```

- Edit `cluster-name` in the downloaded yaml file

```yaml
spec:
    containers:
    - args:
        - --cluster-name=eks-demo # Set the created cluster name
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.4.4
```

- Delete the `ServiceAccount` yaml spec (below) because it's been already created.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
```

- Deploy AWS Load Balancer Controller

```sh
kubectl apply -f v2_4_4_full.yaml
```

- Check if it's deployed successfully

```sh
# Check the deployment
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check the service account
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

