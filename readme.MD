---
Pre-Requisite
---
Install required tools below into your local system
- `kubectl` installed   : https://v1-16.docs.kubernetes.io/docs/tasks/tools/install-kubectl/
- `awscli` installed    : https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
  - Run `aws configure`
  - Fill access key with yout aws account or use another way to connect your aws account
- `eksctl` installed    : https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

---
Main
---
- Create `VPC` for your `EKS`
  - Go to `CloudFormation` service
  - Create new `Stack`
  - Use `CloudFormation` template : https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-09/amazon-eks-vpc-sample.yaml
  - Configure `Subnet` for `Modify auto-assign IP setting` : https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip
    - Go to `VPC` service
    - Click `Subnets`
    - Edit every your `EKS's` `Subnet` 
    - Enable `Modify auto-assign IP setting`

- Create `IAM` Roles for EKS
  - Go to `IAM` service
  - Create role
  - Choose `EKS` then `EKS Cluster`

- Create `EKS` cluster
  - Go to `EKS` service
  - Create new cluster
  - Choose `Role` with `IAM Role` your created before
  - Choose `VPC` with `VPC` your `Stack` created before
  - Choose `Security Group` with `Security Group` your `Stack` created before
  - `Enable logging` -> optional

- Configure your `Subnet` tags
  - Go to `VPC` service
  - Choose `Subnets` linked with your `EKS` cluster
  - Edit your `Subnet` tags as mention below
```
  ------- For public subnets ------
    Key   : kubernetes.io/cluster/<CLUSTERNAME>
    Value : shared

    Key   : kubernetes.io/role/elb
    Value : 1

  ------ For private subnets ------
    Key   : kubernetes.io/role/internal-elb
    Value : 1
```

- Connect local `kubectl` to connect your `EKS` clusrer
  - Wait until your `EKS` cluster is `Active`
  - Run `aws eks --region <region> update-kubeconfig --name <cluster-name>`, for connecting `kubectl` to connect your `EKS` cluster
    - Note: If your having problem for these output is `NoneType object is not iterateable` please follow below step, if you don't please ignore it and continue step
    - You should delete folder `.kube` in `/Users/<user-name>/.kube` for windows user and run command above again
    - For not windows user, please find `kubectl` `.kube` folder in your local system installed and delete it, then run command above again.
  - Run `kubectl get svc` for ensuring kubectl get connected to your `EKS` cluster

- Create Node Group from `EKS` Cluster
  - Go to `IAM` service
  - Create new `IAM` Roles for node cluster: https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role
    - Choose `EC2`, then choose below policies
      - AmazonEKSWorkerNodePolicy
      - AmazonEC2ContainerRegistryReadOnly
      - AmazonEKS_CNI_Policy
  - Go to `EKS` service
  - Choose your `EKS` cluster
  - Choose `Configuration` and Create new `Node Group`  
  - Create New Node Group in `EKS`  
  - Choose `Role` with `IAM Role` you created before
  - Choose the `EC2` spesification for your `EKS` cluster workers and `Scaling` configuration
    - Max     : Maximal scale out instance count
    - Min     : Minimal scale in instance count
    - Desired : Initial instance count
  - Once `Node Group` is `Active`, Go to `EC2` service
    - Check running instance, theres running instances for your `EKS` cluster, usually doesn't have name
    - Enable cloudwatch metric to `auto scaling group` -> optional
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

---
External DNS : https://www.padok.fr/en/blog/external-dns-route53-eks
---
1. Register or transfer domain to Route53
2. Set identity provider `aws eks describe-cluster --name <CLUSTER_NAME> --query “cluster.identity.oidc.issuer” --output text`
3. Add policies, add to NODE-ROLE
```
  {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "route53:ChangeResourceRecordSets"
          ],
          "Resource": [
            "arn:aws:route53:::hostedzone/*"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "route53:ListHostedZones",
            "route53:ListResourceRecordSets"
          ],
          "Resource": [
            "*"
          ]
        }
      ]
    }
```
4. Install External dns with bitnami -> `helm repo add bitnami https://charts.bitnami.com/bitnami`
5. `helm install <RELEASE_NAME> bitnami/external-dns --set provider=aws --set domainFilters[0]=<DOMAIN_FILTER> --set policy=sync --set registry=txt --set txtOwnerId=<HOSTED_ZONE_ID> --set interval=3m -n kube-system` optional `--set rbac.create=true --set rbac.serviceAccountName=<RELEASE_NAME>-external-dns --set rbac.serviceAccountAnnotations.eks\.amazonaws\.com/role-arn=<ROLE_ARN>`
  - RELEASE_NAME - name of the helm release, can be anything you want (external-dns for example)
  - DOMAIN_FILTER - name of your Route53 hosted zone, if *.example.com would be example.com. You can find this information in the AWS console (Route53)
  - HOSTED_ZONE_ID - id of your hosted zone in AWS. You can find this information in the AWS console (Route53)
  - ROLE_ARN - ARN of the role you created earlier in the tutorial
6. Add Ingress resource with annotation `external-dns.alpha.kubernetes.io/hostname: myservice.example.com ` 
- Debugging with `kubectl logs -n kube-system <EXTERNAL_DNS_POD_NAME>`

---
AWS Certificate Manager
---
1. Create public
2. DNS issue -> on validate page auto create
3. Until `issued` state, Add annotate `alb.ingress.kubernetes.io/certificate-arn:<arn-certificate>` & `alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'`
--------------------------------------------------------------------------------------------------------

- source: https://logz.io/blog/amazon-eks-cluster/
- source ingress: https://artifacthub.io/packages/helm/aws/aws-load-balancer-controller
- source 1 ingress multiple namespace:
  1. https://www.padok.fr/en/blog/application-load-balancer-aws
  2. https://stackoverflow.com/questions/58739513/google-kubernetes-engine-how-to-define-one-ingress-for-multiple-namespaces

-------------------------------------------------------------------------------------------------------- 
- custom ingress controller -> use annotate
- Helm
--------------------------------------------------------------------------------------------------------
