# Managing Persistent Storage with Amazon EFS on Amazon EKS

### **1. Download the IAM policy:**
```bash
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

```

### **2. Create an IAM policy:**
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
### **Deploy the Amazon EFS CSI driver