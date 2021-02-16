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
3. Create `EKS` cluster
  - Use `VPC`, `IAM`, and `Security Group` added before
4. Once cluster is `Active`
  - Run `aws eks --region <region> update-kubeconfig --name <cluster-name>` -> tell `kubectl` to connect `EKS`
  - Note: if output is `NoneType object is not iterateable`, you should delete folder `.kube` in `/Users/<user-name>/.kube` and run command above again
  - Run `kubectl get svc` for ensuring kubectl get connected tok `EKS`
5. Create Node Group from `EKS` Cluster
  - Configure `subnet` for `auto assign ipv4`
  - Create `IAM` Roles for node cluster
    1. AmazonEKSWorkerNodePolicy
    2. AmazonEC2ContainerRegistryReadOnly
    3. AmazonEKS_CNI_Policy
    - source: https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role
    - additional source: https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
  - Create New Node Group in `EKS`  

  
--------------------------------------------------------------------------------------------------------

source: https://logz.io/blog/amazon-eks-cluster/
