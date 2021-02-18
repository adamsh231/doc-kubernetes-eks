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

  
--------------------------------------------------------------------------------------------------------

source: https://logz.io/blog/amazon-eks-cluster/
additional source: https://www.padok.fr/en/blog/application-load-balancer-aws (ingress)
