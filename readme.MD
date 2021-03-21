---
Pre-Requisite
---
Install required tools below into your local system
- `kubectl` installed   : https://v1-16.docs.kubernetes.io/docs/tasks/tools/install-kubectl/
- `awscli` installed    : https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
  - Run `aws configure`
  - Fill access key with yout aws account or use another way to connect your aws account
- `eksctl` installed    : https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
- `helm` installed      : https://helm.sh/docs/intro/install/
- `helmfile` optional   : `scoop install helmfile`

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
    - ***Note*** : If your having problem for these output is `NoneType object is not iterateable` please follow below step, if you don't please ignore it and continue step
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
- Create `oidc provider` for your `EKS`
  - Run `eksctl utils associate-iam-oidc-provider --region <AWS_REGION> --cluster <CLUSTER_NAME> --approve`
  - Check in `IAM` service, then go to `Identity provider`
- Download json `Policy` file, Run `curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json`
  - ***Note*** : the name of the file can be cuztomized
- Create new `Policy` for your pod ALB Controller can control `ELB` service, Run `aws iam create-policy --policy-name <POLICY_NAME> --policy-document file://iam-policy.json`
  - Check in your CREATED `Policy` in `IAM` service, then go to `Customer managed policies`
- Create new `Role` for your ALB Controller, Run `eksctl create iamserviceaccount --cluster=<CLUSTER-NAME> --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/<POLICY_NAME> --approve`
  - ***Note*** : if you have run this command twice or more, you should delete the stack first in `CloudFormation`.
- Install and Configure ALB Controller
  - Run `helm repo add eks https://aws.github.io/eks-charts`
  - Run `helm repo update`
  - Run `kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"`
  - Run `helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<CLUSTER_NAME> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller`
- You can add ingress resource now
  - ***WARNING*** : ALB Controller just read Service with `NodePort` not `CLusterIP` -> https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1695
  - Dont forget to annotate as mention below : https://kubernetes-sigs.github.io/aws-load-balancer-controller/guide/ingress/annotations/#annotations
  ```
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/scheme: internet-facing
  ```
---
External DNS : https://www.padok.fr/en/blog/external-dns-route53-eks
---
- Register or transfer domain to Route53 -> ***Pre-Requisite***
- Set identity provider `aws eks describe-cluster --name <CLUSTER_NAME> --query “cluster.identity.oidc.issuer” --output text`
  - ***Note*** : if you have set `oidc` for your cluster please continue, and ignore command above
- Create new `Policy`
  - Go to `IAM` service and Create new `Policy`
  - Fill `Policy` json with below text
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
  - Once `Policy` created
  - Attach `Policy` to the `IAM Role` your worker node
- Install External dns with bitnami
  - Run `helm repo add bitnami https://charts.bitnami.com/bitnami`
  - Run `helm install <RELEASE_NAME> bitnami/external-dns --set provider=aws --set domainFilters[0]=<DOMAIN_FILTER> --set policy=sync --set registry=txt --set txtOwnerId=<HOSTED_ZONE_ID> --set interval=3m -n kube-system`
    - RELEASE_NAME    : Name of the helm release, can be anything you want (external-dns for example)
    - DOMAIN_FILTER   : Name of your Route53 hosted zone, if `*.example.com` would be `example.com`. You can find this information in the AWS console (Route53)
    - HOSTED_ZONE_ID  : ID of your hosted zone in AWS. You can find this information in the AWS console (Route53)
- You can add ingress resource now
  - Add host in ingress
  - Add certificate and listening port and another annotation as mention below : https://kubernetes-sigs.github.io/aws-load-balancer-controller/guide/ingress/annotations/#annotations
  ```
  alb.ingress.kubernetes.io/certificate-arn : arn:aws:acm:xxxx:xxxxx:certificate/xxxxx
  alb.ingress.kubernetes.io/listen-ports    : '[{"HTTP": 80}, {"HTTPS": 443}]'
  ```
- Debugging pod with `kubectl logs <EXTERNAL_DNS_POD_NAME> -n kube-system`

---
AWS Certificate Manager
---
- Go to `Certificate Manager` Service
- Request a `Public Certificate`
- Fill required form
- Use DNS issue
- On validate page, create record
- Once `issued`, copy arn and paste to annotation as mention earlier

---
Pull private registry -> Gitlab
---
Option (1. User Credential, 2. Personal Access Token, 3. Deploy token)
- User Credential
  - `kubectl create secret docker-registry regcred --docker-server=registry.gitlab.com --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email> -n <namespace>`
- Personal Access Token & Deploy Token
  - Create PAT in `Edit Profile` section || All Repo Access -> Read Registry
  - Create DT in `Setting` -> `CI/CD` -> `Repository` || Spesific Repo Access -> Read Registry
  - `kubectl create secret docker-registry regcred --docker-server=registry.gitlab.com --docker-username=<your-name> --docker-password=<your-pword> -n <namespace>`
- Into pod level yaml, change `ImagePullSecrets`
```
imagePullSecrets: 
  - name: gitcred
```
source: https://stackoverflow.com/questions/59456507/kubernetes-gitlab-how-to-store-password-for-private-registry

------
Development Only
--------------------------------------------------------------------------------------------------------
- custom ingress controller -> use annotate
- Auto redirect -> https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/tasks/ssl_redirect/ or Manually in `ALB`
- Cloudfront -> https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html#GettingStartedSignup -> ACM only us-east-1 N-Virginia
- Route Migrate -> https://lobster1234.github.io/2017/05/10/migrating-a-domain-to-amazon-route53/
--------------------------------------------------------------------------------------------------------
