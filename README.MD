# Migrate Your Existing OnPremise Workloads to Amazon EC2

Using `VM Import` migrate your existing VM-based applications and preserve the software and settings that you have configured in your existing VM.

![Fig : Migrate Your Existing Applications and Workloads to Amazon EC2](https://raw.githubusercontent.com/miztiik/server-migration-onprem-to-aws/master/images/miztiik-server-migration-onprem-to-aws-2.png)

#### Follow this article in [Youtube](https://youtu.be/buzusNljpy4)

0. ## Prerequisites

    - OnPremise VM (_Preferably in VMWare / Virtualbox_)
        - If you have `*.vmdk` image of your VM that will also be enough
        - **MUST**: You should have the `uid`/`password` to log into this VM
    - AWS CLI with access to `Administrator` privileges
        - _You can tighten it down based on your requirements_

1. ## Export VM & Upload to S3

    Depending on virtualization tool, use the appropriate procedure to export your VM into `*.vmdk` or `*.ova` image. Upload the image to S3 Bucket and note down the `bucket_name` and `vm_image_name`.

1. ## Global Customization Variables

    ```sh
    bucket_name="n-backup"
    # Add the appropriate S3 Prefix to the VM Image
    vm_image_name="VM-Import/vCentOS7-disk002.vmdk"
    ```

1. ### Create Trust Policy

    Create the IAM trust policy json with the name `trust-policy.json`

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

1. ### Create the IAM Role for VM Import

    Ensure that you create the role with the name `vmimport`. Use the trust policy created in the previous step

    ```sh

    aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
    ```

1. ### Create the IAM Policy: `role-policy.json`

    This policy will be attached to the role `vmimport` created in the previous step. The bucket name is picked up from the `global variable`.

    ```sh
    echo '{
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
                "arn:aws:s3:::'${bucket_name}'/*"
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

1. ### Attach policy to IAM Role:`vmimport`

    ```sh
    aws iam put-role-policy --role-name vmimport \
                            --policy-name vmimport \
                            --policy-document "file://role-policy.json"
    ```

1. ## Begin VM Image Import Task

    The following command will begin the import of the VM Image. The S3 Bucket name & Bucket Key is picked up from the global variables.

    ```sh
    # Set the metadata, 
    echo '[
      {
        "Description": "centosv7",
        "Format": "vmdk",
        "UserBucket": {
            "S3Bucket": "'${bucket_name}'",
            "S3Key": "'${vm_image_name}'"
        }
    }]
    ' > containers.json
    ```

    ###  Begin VM Import

      ```sh
      aws ec2 import-image --description "centosv7" --disk-containers "file://containers.json"
      ```

      _The expected output,_

      ```sh
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
      ```

      Note down the `ImportTaskId` to check the progress of the import job.

    ### Check status of VM Import Jobs

      ```sh
      aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d6db3a35d431e4e3"
      ```

    ### Check VM Import Progress

      ```sh
      # VM Image being updated to AMI
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
      ```

    ### Completion Status

      ```sh
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

1. ## Launch New EC2

    Once you launch the VM, you can login using the same `uid`/`password` you used onpremise. Typically in real-world you will clean this before the import task and setup SSH key-based authentication

## Support

Please open a [GitHub issue](https://github.com/miztiik/server-migration-onprem-to-aws/issues/new).

## Feedback

Please open a [GitHub issue](https://github.com/miztiik/server-migration-onprem-to-aws/issues/new). I encourage you to contribute your changes and send me pull request.

##### References

1. [AWS CLI - import-image](https://docs.aws.amazon.com/cli/latest/reference/ec2/import-image.html)

1. [AWS Blogs - Importing a VM](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-vm)
