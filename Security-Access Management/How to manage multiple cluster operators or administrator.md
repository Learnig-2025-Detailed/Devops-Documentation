

## How to manage multiple cluster operators or administrator?

## Deprecated aws-auth ConfigMap method for Access Control
AWS in this Access control mechanism relied to authorize administrative access to EKS Cluster.
Issues with this AC
-	Authentication occurs in AWS IAM
-	Authorization takes place in K8s
This require cluster admin to switch between AWS IAM and K8s APIs and configuration files when managing multiple cluster operators <this is error prone>

New Cluster Access Management
We can automatically associate an IAM principal with new predefined AWS-managed K8s policies known as access policies and EKS will do all the behind-the scene mapping between policies and K8s RBAC groups.
ACCESS ENTRIES AND POLICY ASSOCIATION
This section introduces the steps involved in creating cluster administrator with new EKS cluster access management method
In this method, it encompasses a three-step process
-	Creation of IAM Principal in EKS IAM (access entry)
-	The association of EKS principal with access policies or custom K8s permissions (access policy association)
-	Automated behind the scene mapping
ACCESS ENTRY 
Is where a cluster administrator registers an AWS IAM principal in AWS/EKS access entry IAM Service.
ACCESS POLICY STEP
A Cluster admin attaches policies to the registered AWS/EKS IAM principal to access the K8s components and objects. This Binding step involves attachment of access policies to an IAM Principal.
BUILT-IN ACCESS ENTRY POLICY
Command to find managed policies - aws eks list-access-policies –output table
Default K8s Role	Privileges	AWS EKS Policy	Administrator
Cluster-Admin	Cluster-wide super user	AmazonEKClusterAdminPolicy	Cluster Creator
Admin	Full Access within a namespace	AmazonEKSAdminPolicy	Namespace Admin
Edit	Read/Write within a namespace	AmazonEKSEditPolicy	Namespace editor
View	Read-only with a namespace	AmazonEKSViewPolicy	Security Auditor
Custom	Cluster Developer	Custom Policy	Dev/Admins

NOTE: AmazonEKClusterAdminPolicy – all other policy is scoped within Namespace. Namespace Admin manages resources and access to the Namespace.
ACCECC MANAGEMENT AUTHENTICATION MODES
Backward compatible, can be hybrid also. Three modes of operation:
-	API, API_AND_CONFIG_MAP AND CONFIG_MAP
COMMAND TO SWITCH AUTHENTICATION MODE AFTER CLUSTER CREATED:
aws eks update-cluster-config --name <cluster_name> --access-config authentication mode=API or 
API_AND_CONFIG_MAP
Switching modes = CONFIG_MAP  API_AND_CONFIG_MAP  API , not allowed = API  API_AND_CONFIG_MAP or CONFIG_MAP , cannot revert API_AND_CONFIG_MAP  CONFIG_MAP
IMPLEMENTATION OF NEW ACCESS MANAGEMENT

NOTE: bootstrap cluster admin permission=true means grant cluster creator full access to cluster
access_config {
    authentication_mode = "API_AND_CONFIG_MAP"
   bootstrap_cluster_creator_admin_permissions = "True"
  }


For Implementing various access control, create user Alice.
IMPLEMENTATION 1: AUTHENTICATION MODE = API AND CONFIG_MAP
1.	Deploy cluster and switch context to default cluster creator
aws eks update-kubeconfig --name <CLUSTER_NAME>
2.	List available access entires for current user
aws eks list-access-entires –cluster-name <CLUSTER_NAME>
3.	Create user – Alice
aws iam create-user --user-name alice
4.	Create policy file:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "*"
        }
    ]
}

NOTE: eks:DescribeCluster PERMISSION MANADATORY FOR ALL CLUSTER ADMINS.

5.	Create an IAM Policy with the policy.json file above
aws iam  create-policy –policy-name eks-describe-pilicy --policy-document file://policy.json 
6.	ATTACH EKS DESCRIBE CLUSTER POLICY TO USER and copy ARN for alice
aws iam attach-user-policy –policy-arn arn:aws:iam::$(ACCOUNT_ID):policy/eks-describe-policy –user-name alice
7.	Create Access entry for alice
aws eks create-access-entry –cluster-name <CLUSTER_NAME> --principal-arn <IAM_PRINCIPAL_ARN>
aws eks create-access-entry –cluster-name <CLUSTER_NAME> --principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice
8.	View available access policies
aws eks list-access-policies –output table
9.	Assign cluster Creator Policy to alice
aws eks associate-access-policy\
--cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice \
--policy_arn arn:aws:eks::cluster-access-policy/AmazonEKSClusterAdminPolicy \
--access-scope type=cluster

access-scope type=cluster grant access to Alice on all namespaces.


To Test:
Create Alice profile on infra server or local – 

[alice]
aws_access_key_id = <YOUR_AWS_ACCESS_KEY_ID>
aws_secret_access_key = >YOUR_AWS_SECRET_ACCESS_KEY>
region = AWS_REGION

10.	Switch user context
aws eks update-kubeconfig --name <CLUSTER_NAME> --profile alice

IMPLEMENTATION 2: Make Alice default namespace admin
1.	switch context to default cluster creator
2.	Remove cluster Creator access for Alice 
aws eks disassociate-access-policy\
--cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice \
--policy_arn arn:aws:eks::cluster-access-policy/AmazonEKSClusterAdminPolicy \

3.	Downgrade Alice permission to a namespace Admin
aws eks associate-access-policy\
--cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice \
--policy_arn --principal-arn arn:aws:eks::cluster-access-policy/AmazonEKSAdminPolicy \
--access-scope type=cluster

Observation : kubectl get pods -A will fail.

IMPLEMENTATION 3: Make Alice specific namespace admin
aws eks associate-access-policy\
--cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice \
--policy_arn --principal-arn arn:aws:eks::cluster-access-policy/AmazonEKSAdminPolicy \
--access-scope type=namespace,namespaces=test-ns


IMPLEMENTATION 4: Grant a Custom developer permission to Alice through Kubernetes Group.
1.	switch context to default cluster creator
2.	Create Cluster Role Binding to Alice
kubectl create clusterrolebinding readonly \
--clusterrole=readonly-role\
--group=readonlt-group

3.	Define Role permission and binding with below yaml

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata: 
  name: readonly-role
rules:
-	apiGroups: [“”]
  resources: [“nodes”,”namespaces”,”pods”]
  verbs: [“get”, “list”]

-	apiGroups: [“apps”]
   resources: [“deployment”, “deamonset”]
   verbs: [“get”, “list”, “watch”]

4.	Execute binding 
kubectl apply -f rolebinding.yaml

5.	Remove access entry for alice
aws eks delete-access-entry --cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice

6.	Create new access entry for alice
aws eks create-access-entry --cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):user/alice \
--kubernetes-group readonly-group

7.	Switch context to Alice and test 


IMPLEMENTATION 4: Assign IAM Permission to a role rather than an IAM User. Used when we want to assign same permission to several users.
1.	switch context to default cluster creator
2.	create eks-describe-policy
3.	create trusted relationship policy file trust.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::980305500578:user/EKSClusterUser"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

4.	Create Role with trusted relationship policy and copy ARN
aws iam create-role –role-name my-test-eks-role \
--assume-role policy-document file://truct.json

5.	Attach EKS cluster Describe policy to above role
aws iam attach-role-policy --role-name my-test-eks-role --policy-arn arn:aws:iam::980305500578:policy/eks-describe-policy

6.	Create new profile for alice in .aws/credentials file
[alice]
aws_access_key_id = <YOUR_AWS_ACCESS_KEY_ID>
aws_secret_access_key = >YOUR_AWS_SECRET_ACCESS_KEY>
region = aws-region

[alice-role]
Role_arn = arn:aws:iam::980305500578:role/my-test-eks-role
Source_profile = alice
Region = us-east-1

7.	Add entry for the role so that the user who assume the role will become cluster admin.
aws eks create-access-entry –cluster-name <CLUSTER_NAME> --principal-arn arn:aws:iam::$(ACCOUNT_ID):role/my-test-eks-role

8.	Create permission with role ARN
aws eks associate-access-policy\
--cluster-name <cluster_name> \
--principal-arn arn:aws:iam::$(ACCOUNT_ID):role/my-test-eks-role \
--policy_arn arn:aws:eks::cluster-access-policy/AmazonEKSClusterAdminPolicy \
--access-scope type=cluster

9.	Switch Context
Aws eks update-kubeconfig --name <cluserter name> --profile alice-profile
