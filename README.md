# Managing Persistent Storage with Amazon EFS on Amazon EKS

1. Download the IAM policy:
```bash
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

```
2. Create an IAM policy:
```bash
aws iam create-policy 
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy 
    --policy-document file://iam-policy-example.json
```
### **3. To determine your cluster's OIDC provider ID, run the following command:**
```bash
aws eks describe-cluster --name your_cluster_name --query "cluster.identity.oidc.issuer" --output text
```
**Note:** Replace your_cluster_name with your cluster name.

### **4. Create the following IAM trust policy, and then grant the AssumeRoleWithWebIdentity action to your Kubernetes service account:**
```json
cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF
```
**Note:** Replace **YOUR_AWS_ACCOUNT_ID** with your account ID, **YOUR_AWS_REGION** with your AWS Region, and **XXXXXXXXXX45D83924220DC4815XXXXX** with your cluster's OIDC provider ID.

### **5. Create an IAM role:**
```bash
aws iam create-role 
  --role-name AmazonEKS_EFS_CSI_DriverRole 
  --assume-role-policy-document file://"trust-policy.json"
```
### **6. Attach your new IAM policy to the role:**
```bash
aws iam attach-role-policy 
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy 
  --role-name AmazonEKS_EFS_CSI_DriverRole
```
### **7. Attach your new IAM policy to the role:**
```bash
aws iam attach-role-policy 
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy 
  --role-name AmazonEKS_EFS_CSI_DriverRole
```
### **8. Save the following contents to a file that's named efs-service-account.yaml:**
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: aws-efs-csi-driver
  name: efs-csi-controller-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/AmazonEKS_EFS_CSI_DriverRole
```
### **9. Create the Kubernetes service account on your cluster:**
```bash
kubectl apply -f efs-service-account.yaml
```
**Note:** The Kubernetes service account that's named efs-csi-controller-sa is annotated with the IAM role that you created.
### **10. Download the manifest from the public Amazon ECR registry and use the images to install the driver:**
```bash
$ kubectl kustomize "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.5" > public-ecr-driver.yaml
```
**Note:** To install the EFS CSI driver, you can use Helm and a Kustomize with AWS Private or Public Registry. For instructions on how to install the EFS CSI Driver, see the AWS EFS CSI driver on the GitHub website.

### **11.** Edit the file **public-ecr-driver.yaml** and annotate **efs-csi-controller-sa** Kubernetes service account section with the IAM role's ARN:
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: aws-efs-csi-driver
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<accountid>:role/AmazonEKS_EFS_CSI_DriverRole
  name: efs-csi-controller-sa
  namespace: kube-system
```
### **12. Deploy the Amazon EFS CSI driver**
```bash
kubectl apply -f public-ecr-driver.yaml
```
### **[ Helm ]**
This procedure requires Helm V3 or later. To install or upgrade Helm, see [Using Helm with Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html).  
**To install the driver using Helm**
#### 1. Add the Helm repo.
```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```
#### 2. Update the repo.
```bash
helm repo update aws-efs-csi-driver
```
#### 3. Install a release of the driver using the Helm chart.
```bash
helm upgrade --install aws-efs-csi-driver --namespace kube-system aws-efs-csi-driver/aws-efs-csi-driver
```
### **Create an Amazon EFS File system**
1. To get the virtual private cloud (VPC) ID of your Amazon EKS cluster, run the following command:
```bash
To get the virtual private cloud (VPC) ID of your Amazon EKS cluster, run the following command:
```
**Note:** Replace **your_cluster_name** with your cluster name.
2. To get the CIDR range for your VPC cluster, run the following command:
```bash
aws ec2 describe-vpcs --vpc-ids YOUR_VPC_ID --query "Vpcs[].CidrBlock" --output text
```
**Note:** Replace the **YOUR_VPC_ID** with your VPC ID.
3. Create a security group that allows inbound network file system (NFS) traffic for your Amazon EFS mount points:
```bash
aws ec2 create-security-group --description efs-test-sg --group-name efs-sg --vpc-id YOUR_VPC_ID
```
**Note:** Replace **YOUR_VPC_ID** with your VPC ID. Note the **GroupId** to use later.
4. To allow resources in your VPC to communicate with your Amazon EFS file system, add an NFS inbound rule:
```bash
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 2049 --cidr YOUR_VPC_CIDR
```
**Note:** Replace **YOUR_VPC_CIDR** with your VPC CIDR and **sg-xxx** with your security group ID.
5. Create an Amazon EFS file system for your Amazon EKS cluster:
```bash
aws efs create-file-system --creation-token eks-efs
```
**Note:** Note the **FileSystemId** to use later.
6. To create a mount target for Amazon EFS, run the following command:
```bash
aws efs create-mount-target --file-system-id FileSystemId --subnet-id SubnetID --security-group sg-xxx
```
**Important:** Run the preceding command for all the Availability Zones with the _**SubnetID**_ in the Availability Zone where your worker nodes are running. Replace **FileSystemId** with your EFS file system's ID, **sg-xxx** with your security group's ID, and **SubnetID** with your worker node subnet's ID. To create mount targets in multiple subnets, run the command for each subnet ID. It's a best practice to create a mount target in each Availability Zone where your worker nodes are running. You can create mount targets for all the Availability Zones where worker nodes are launched. Then, all the Amazon Elastic Compute Cloud (Amazon EC2) instances in these Availability Zones can use the file system.
### **Test the Amazon EFS CSI driver**
