# Managing Persistent Storage with Amazon S3 on Amazon EKS

### **1. Download the IAM policy:**
```bash
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

```

### **2. Create an IAM policy:**
````bash
aws iam create-policy 
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy 
    --policy-document file://iam-policy-example.json
    
```