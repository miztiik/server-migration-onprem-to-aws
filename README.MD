

## Global Customisation Variables
bucket_name="n-backup"


### Create Trust Policy
```json
cat > "trust-policy.json" << "EOF"
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
EOF
```
### Create the Role
```sh
aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
```

### Create the `role-policy.json`
```json
echo echo '{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket" 
         ],
         "Resource":[
            "arn:aws:s3:::'${bucket_name}'",
            "arn:aws:s3:::'${bucket_name}/*'"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}
' | sudo tee role-policy.json
```

### Attach Policy to Role
```sh
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"
```

## Begin Image Import
aws ec2 import-image --description "centosv7" --disk-containers "file://containers.json"

--license-type=Auto \
cat > "containers.json" << "EOF"
[
  {
    "Description": "centosv7",
    "Format": "vmdk",
    "UserBucket": {
        "S3Bucket": "n-backup",
        "S3Key": "VM-Import/vCentOS7-disk002.vmdk"
    }
}]
EOF

###### Expected Output
```sh
[root:tmp]#aws ec2 import-image --description "centosv7" --disk-containers "file://containers.json"
{
    "Description": "centosv7",
    "ImportTaskId": "import-ami-0d6db3a35d431e4e3",
    "Progress": "2",
    "SnapshotDetails": [
        {
            "DiskImageSize": 0.0,
            "Format": "VMDK",
            "UserBucket": {
                "S3Bucket": "n-backup",
                "S3Key": "VM-Import/vCentOS7-disk002.vmdk"
            }
        }
    ],
    "Status": "active",
    "StatusMessage": "pending"
}

[root:tmp]# aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d6db3a35d431e4e3"
{
    "ImportImageTasks": [
        {
            "Description": "centosv7",
            "ImportTaskId": "import-ami-0d6db3a35d431e4e3",
            "Progress": "30",
            "SnapshotDetails": [
                {
                    "Description": "centosv7",
                    "DiskImageSize": 931182592.0,
                    "Format": "VMDK",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "n-backup",
                        "S3Key": "VM-Import/vCentOS7-disk002.vmdk"
                    }
                }
            ],
            "Status": "active",
            "StatusMessage": "updating"
        }
    ]
}
[root:tmp]# aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d6db3a35d431e4e3"
{
    "ImportImageTasks": [
        {
            "Architecture": "x86_64",
            "Description": "centosv7",
            "ImageId": "ami-0da97e2296167b5ca",
            "ImportTaskId": "import-ami-0d6db3a35d431e4e3",
            "LicenseType": "BYOL",
            "Platform": "Linux",
            "SnapshotDetails": [
                {
                    "Description": "centosv7",
                    "DeviceName": "/dev/sda1",
                    "DiskImageSize": 931182592.0,
                    "Format": "VMDK",
                    "SnapshotId": "snap-0dc6d32a5924b22c7",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "n-backup",
                        "S3Key": "VM-Import/vCentOS7-disk002.vmdk"
                    }
                }
            ],
            "Status": "completed"
        }
    ]
}

```

## Status of Import Jobs
```sh
aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d6db3a35d431e4e3"
```