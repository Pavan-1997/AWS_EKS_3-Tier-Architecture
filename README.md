# AWS_EKS_3-TIER-ARCHITECTURE

## THREE TIER ARCHITECTURAL MODEL:

- FRONT-END - UI - Presentation Layer

- BACK-END - Logic Layer

- DATABASE - Data Layer


## HIGH LEVEL DESIGN:

- Workflows -> Components

- User > Registration, Login > Catalogue - AI, Robots > Rating for Robot > Cart > Payments - Integrated with any payment gateway > Shipping details > Order completed user gets a Order ID

- We can do it as a Monolithic or Microservice architechture 

- UI -> Angular (Presentation)

- Logic -> Microservice (Cart, Catalogue, Payments, Shipping, User)

- DB -> Redis (In memory datastore), MongoDB (User details), MySQL (Ratings)

- OIDC - Integrating service accounts of Persistant volumes to IAM roles/policies (Redis -> Stateful Set -> Persistant Volume -> EBS)

---

1. Now goto EC2 from AWS Console -> Click on Launch instance

	Give a name
	
	Use Ubuntu as an image
	
	Instance type as t2.micro
	
	Create a key pair if already present use existing one
	
	Click on Launch instance
	
	Connect to the instance


2. Install Docker
```
sudo apt update -y

sudo apt install apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"

apt-cache policy docker-ce

sudo apt install docker-ce

sudo usermod -aG docker $USER

sudo systemctl status docker 
```
```
sudo reboot
```
`Reboot the instance for Ubuntu user to execute docker commands`




3. Install AWS CLI
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

sudo apt install unzip

unzip awscliv2.zip

sudo ./aws/install

aws --version
```

4. Configure AWS CLI
```
aws configure
```
`Just give Access Key and Secret Key followed by ENTER-ENTER`

`REGION - us-west-1`


5. Install Kubectl
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

kubectl version
```

6. Install Eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp 

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

7. Create a EKS Cluster on AWS without NodeGroup
```   
eksctl create cluster --name=eks-robot-shop-server --region=us-west-1 --zones=us-west-1a,us-west-1c --without-nodegroup
```

8. Create a NodeGroup with 2 instances giving necessary access with a existing keypair
```
eksctl create nodegroup --cluster=eks-robot-shop-server --region=us-west-1 --name=eksdemo-ng-public --node-type=t2.medium --nodes=2 --nodes-min=2 --nodes-max=4 --node-volume-size=10 --ssh-access --ssh-public-key=AWS-KEYPAIR-NC --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access
```

9. Configure IAM OIDC provider
```
export cluster_name=eks-robot-shop-server

oidc_id=$(aws eks describe-cluster --region us-west-1 --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

10. Check if there is an IAM OIDC provider configured already
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If not, run the below command
```
eksctl utils associate-iam-oidc-provider --region us-west-1 --cluster $cluster_name --approve
```

11. a) Setup ALB add-on by Download IAM policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

b) Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
	
c) Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=$cluster_name \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::009403810934:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

d) Install Helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
```
```
chmod +x get_helm.sh
./get_helm.sh
```
```
helm version
```

e) Add helm repo
```
helm repo add eks https://aws.github.io/eks-charts
```

f) Update the repo
```
helm repo update eks
```

g) Install helm
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$cluster_name --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-west-1 --set vpcId=vpc-080eba0d104598a17
```

h) Verify that the deployments are running.
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

11. a) `EBS CSI - when a PVC is created then the EBS volume is automatically created and attached to the Redis Stateful Set`

Create an IAM role and attach a policy. AWS maintains an AWS managed policy or you can create your own custom policy. You can create an IAM role and attach the AWS managed policy with the following command. Replace my-cluster with the name of your cluster. The command deploys an AWS CloudFormation stack that creates an IAM role and attaches the IAM policy to it.

```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $cluster_name \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```
	
b) Create CSI Add-on
```
eksctl create addon --name aws-ebs-csi-driver --cluster $cluster_name --service-account-role-arn arn:aws:iam::009403810934:role/AmazonEKS_EBS_CSI_DriverRole --force
```

12. Create a namespace
```
kubectl create ns robot-shop
```
		
13. Clone the repo
```
git clone https://github.com/Pavan-1997/AWS_EKS_3-Tier-Architecture.git 
```

14. Install helm
```
cd /home/ubuntu/AWS_EKS_3-Tier-Architecture/three-tier-architecture-demo/EKS/helm/
```
```
helm install robot-shop --namespace robot-shop .
```

15. Verify the pods
```
kubectl get pods -n robot-shop
```

16. Verify the service
```
kubectl get svc -n robot-shop
```
You can see the LB


17. Using Ingress
```
kubectl apply -f ingress.yaml
```

18. Verify the service
```
kubectl get svc -n robot-shop
```
You can see the Ingress

19. Now go to the AWS EC2 Load Balancers and access the DNS


