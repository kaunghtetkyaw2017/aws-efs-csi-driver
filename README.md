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
