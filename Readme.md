Pre-requisites:																		kubernetes.io/role/elb:1
==============
1. Create an EKS Cluster
2. Create an IAM Identity and Access Management (IAM) OpenID Connect (OIDC) provider for your cluster
3. Create a Policy for the EKS OIDC
4. Create a Role for the EKS OIDC with the above policy
5. Create a ServiceAccount 
6. Install AWS Load Balancer Contoller Using HELM


#### AWSCLI
```
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    sudo apt install unzip
    unzip awscliv2.zip
    sudo ./aws/install
```

#### Terraform
```
    sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

    wget -O- https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list

    sudo apt update

    sudo apt-get install terraform -y

```

#### Kubectl
```
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/
```

#### eksctl
```
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
```

#### Helm
```
	curl -LO https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
	tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
	sudo mv linux-amd64/helm /usr/local/bin/helm
	export PATH=$PATH:/usr/local/bin
	helm version
```

#### Update KubeConfig File
```
    aws eks --region ap-south-1 update-kubeconfig --name devops-cluster
```
	
Doc Ref: https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html

#### Step-1: EKS Cluster Setup
	- The code is in the GitHub Repo: https://github.com/Venkat3699/EKS_Terraform.git
	
#### Step-2: IAM OpenID Connect	
	- Copy the EKS Cluster OIDC URL (Click on EKS Cluster Name --> Overview -> OpenID Connect provider URL)
	- Go to IAM Console
	- Click on Identity provider:
		- Click on Add provider	
		- Select OpenID connect 
		- Provide URL which is copied from EKS 
		- Audience: sts.amazonaws.com
		- click on Add provider
	
#### Step-3: Policy Creation		
	- (Another URL: https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/v2.12.0/docs/install/iam_policy.json)

	- The Policy is in this URL: https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.12.0/docs/install/iam_policy.json

	- Go to IAM Console
	- Click on policies:
		- Click on Create policy
		- Select Json format
		- Paste the policy which is copied from above URL
		- Click on Next
		- policy Name: albController
		- Click on create policy
		
#### Step-4: Role Creation
	- Go to IAM Console
	- Click on Roles:
		- click on Create Role
		- Select Web Identity
		- Identity provider: select our above created Identity Provider
		- Audience: sts.amazonaws.com
		- Click on Next
		- Select the Custom Created policy: albController
		- Click on Next
		- Role Name: EKS-albControllerRole
		- Click on Create Role
		
	- Click on the Role Name: EKS-albControllerRole
		- Click on Trust Relationship
		- Click on edit trust policy
		- Add this line: (Make sure to check OIDC code: A20B838FB661493BA9F9CE719C9A8021)
			"oidc.eks.ap-south-1.amazonaws.com/id/A20B838FB661493BA9F9CE719C9A8021:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
		- Click on Update Policy
		- Copy the Role ARN (arn:aws:iam::682033479178:role/EKS-albControllerRole)

#### Step-5: Service Account Creation on EKS Cluster
```
		kubectl apply -f service_account.yml
		kubectl describe serviceaccount aws-load-balancer-controller -n kube-system (check the role is attached correctly or not)
```		
#### Step-6: Install AWS LB Contoller:
```
	helm repo add eks https://aws.github.io/eks-charts
	helm repo update eks
```
	- Configuring AWS Load Balancer Controller
```
	helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
	   -n kube-system \
	   --set clusterName=my-cluster \	# Change to EKS Cluster Name
	   --set serviceAccount.create=false \
	   --set region=ap-south-1 \
	   --set vpcId=vpc-0975ee9ecfb931a1d \ 		# Change to our EKS Cluster VPC ID
	   --set serviceAccount.name=aws-load-balancer-controller
```
```	   
	kubectl get deployment -n kube-system aws-load-balancer-controller
```	
	- We have successfully created the AWS LB Controller, we can deploy the ingress 
	
#### Step-7: Deploy the Ingress 
	- kubectl apply -f deployment.yml		  
	- kubectl get ingress -A\
	
#### Step-8: Check Load Balancer is Created or Not
	- Go to EC2 console
	- Click on Load Balancer
	- Check the DNS name is same or not
	- Access the application using DNS name
	
#### Step-9: To delete all the resources
	- kubectl delete -f deployment.yml
	- kubectl delete -f service_account.yml
	- helm uninstall aws-load-balancer-controller -n kube-system
	- terraform destroy --auto-approve