---
Pre-Requisite
---
- `kubectl` installed
- `awscli` installed -> auth first using `awscli configure` -> `create new key`
- `eksctl` installed

---
Main
---
1. Create `IAM` Roles
  - EKS -> EKS Cluster
2. Create `VPC` for `EKS`
  - Use `Cloud Formation` -> https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-09/amazon-eks-vpc-sample.yaml
  - Configure `subnet` for `auto assign ipv4` -> https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip
  - Configure your subnet tags as mention below
-----------------
  - ------ For public subnets ------
  - Key: `kubernetes.io/cluster/<CLUSTERNAME>`
    Value: `shared`

  - Key: `kubernetes.io/role/elb`
    Value: `1`

  ------ For private subnets ------
  - Key: kubernetes.io/role/internal-elb
    Value: 1
---------------------
3. Create `EKS` cluster
  - Use `VPC`, `IAM`, and `Security Group` added before
4. Once cluster is `Active`
  - Run `aws eks --region <region> update-kubeconfig --name <cluster-name>` -> tell `kubectl` to connect `EKS`
  - Note: if output is `NoneType object is not iterateable`, you should delete folder `.kube` in `/Users/<user-name>/.kube` and run command above again
  - Run `kubectl get svc` for ensuring kubectl get connected tok `EKS`
5. Create Node Group from `EKS` Cluster
  - Create `IAM` Roles for node cluster
    1. AmazonEKSWorkerNodePolicy
    2. AmazonEC2ContainerRegistryReadOnly
    3. AmazonEKS_CNI_Policy
    - source: https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role
  - Create New Node Group in `EKS`  
    - Add cloudwatch metric to `auto scaling group` -> optional
    - source: http://www.scalingbits.com/aws/cloudformation/amiids -> ami ids
    - Run `kubectl get node` to ensure the node is connected and status id `Ready`

---
Ingress with ALB Controller
---
- `eksctl utils associate-iam-oidc-provider --region <AWS_REGION> --cluster <CLUSTER_NAME> --approve`
- `curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json`
- `aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json`
- `eksctl create iamserviceaccount --cluster=<cluster-name> --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy --approve` --> check cloud formation for duplicate service account (use service account instead of IAM role)
- `helm repo add eks https://aws.github.io/eks-charts`
- `helm repo update`
- `kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"`
- `helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<k8s-cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller`
- Install ingress resource to service just `NodePort` -> https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1695
- Dont forget to annotate https://kubernetes-sigs.github.io/aws-load-balancer-controller/guide/ingress/annotations/#annotations
- `kubernetes.io/ingress.class: alb`
- `alb.ingress.kubernetes.io/scheme: internet-facing`

--------------------------------------------------------------------------------------------------------

- source: https://logz.io/blog/amazon-eks-cluster/
- source ingress: https://artifacthub.io/packages/helm/aws/aws-load-balancer-controller


