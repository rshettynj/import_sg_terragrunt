# import_sg_terragrunt
This is a demo repository for importing and managing the AWS security groups.
##**Purpose**
Lot of security groups may have been created using Clickops. it makes more sense to stop using clickops and bring these resources under Terraform control. Terragrunt is wrapper around Terraform and provides many benefits (individual resource state management and keeping code DRY primarily and many other benefits.)

##**Scenario below:**
We will import 2 security groups and its connected CIDR egress/ingress rules under terragrunt control. Once imported, we will also update the CIDR rules in the existing security groups, modify other attributes.  After that, we will also add new security groups and rules using Terragrunt. No more clickops.
If someone created more security groups again, use the same process to import back to Terragrunt.


#**STORY 1**
##**Importing security groups and CIDR rules.**

Use a desktop or an ec2 instance where Terragrunt/terraform is installed.  I use an AWS Amazon linux instance.  It has permission to manage resources from my personal AWS account using AWS IAM role. This setup detail is not covered here.

```
[root@ip-172-30-2-182 sg3]# terragrunt --version
terragrunt version v0.75.4

[root@ip-172-30-2-182 modules]# terraform --version
Terraform v1.11.0
on linux_amd64
```

Terragrunt uses one atomic directory and only one terragrunt.hcl in it to mange the resources. Actual public module will be stored in the ./modules folder.  We are not covering the terragrunt layout details here.
In this case, we have sg3 folder and ./terragrunt.hcl file in it.

[root@ip-172-30-2-182 sg2]# cat sg3/terrgrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../modules/general_import"
}

inputs = {
}


#Below details are for storing the state file in the AWS s3 bucket and lock file in the AWS dynamodb table.
#Note how we are using aws credentials file in the terragrung config file.  You may use a role instead if needed.  I use the AWS us-east-1 region.
[root@ip-172-30-2-182 terragrunt]# cat root.hcl
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket = "xxxx374nsddhg3223gg2yyy"

    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = "us-east-1"
    profile         = "default"
    shared_credentials_file = "/root/.aws/credentials"
    encrypt        = true
    dynamodb_table = "dddd38787sdksbkdsdffff"

  }
}

generate "provider" {
  path = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents = <<EOF
provider "aws" {
  region = "us-east-1"
  profile = "default"
}
EOF
}

[root@ip-172-30-2-182 terragrunt]# cat /root/.aws/credentials
[default]
aws_access_key_id = 1111YEYSMFUTTCQDQFP6
aws_secret_access_key = 22222BIIbPAHcqE3zXVjVVN7w6gTEUDu+hUcuq

[root@ip-172-30-2-182 sg3]# tree
.
├── terragrunt.hcl

0 directories, 1 file
[root@ip-172-30-2-182 sg3]# cd ../../modules/gene*
[root@ip-172-30-2-182 general_import]# tree
.
├── sg.tf

0 directories, 1 file


Explanation for entries in the ../../modules/general_import/sg.tf file (below)
----------------------------------------
Note: I create a folder called "general_import" and store the below content in the file called sg.tf.  it can be any name ending with .tf.
List out all the security groups and rules (egress/ingress) to be imported as shown.  You could even import 100's of security groups!!

We are using CIDR rules import separate from security group import is because "CIDR rules" are considered separate resources in the Terraform documentation. (AWS VPC rules are different than AWS VPC security groups. each has its own published module or resource block.
Some resources are published as modules and some as just resources.  
Hint: go to https://registry.terraform.io/providers/hashicorp/aws/latest/docs  search for "aws security group" and you will get https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws/latest
click on source code link and open main.tf file and look for the resource name which is "resource "aws_security_group" "this"   
use "aws_security_group".  Subgroup "this" can be anything but must be unique within a tf file. So we use this_1

Similary to find the CIDR rule module or resource go to https://registry.terraform.io/providers/hashicorp/aws/latest/doc search for "aws securty group rule". You will end up at https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group_rule which clearly shows NOT to use that resource but instead use aws_vpc_security_group_egress_rule and aws_vpc_security_group_inress_rule resource. search for these resources and  pickup the example usage resource name that is "aws_vpc_security_group_egress_rule" and "aws_vpc_security_group_inress_rule".

"id" is the security group id (sg-xxxx) and "security group rule id". (sgr-xxx)

cat ../../modules/general_import/sg.tf
# security group 1 and its egress and ingress to import
import {
  to = aws_security_group.this_1
  id = "sg-00740ac44c8417888"
}
import {
  to = aws_vpc_security_group_egress_rule.egress_1
  id = "sgr-0ab442137d13e14444"
}
import {
  to = aws_vpc_security_group_ingress_rule.ingress_1
  id = "sgr-0b47fc73bb4a0b555"
}
# security group 2 and its egress and ingress to import
import {
  to = aws_security_group.this_2
  id ="sg-0b6c55edf0b531123"
}
import {
  to = aws_vpc_security_group_egress_rule.egress_2
  id = "sgr-0036c6bae9ea42222"
}
import {
  to = aws_vpc_security_group_ingress_rule.ingress_2
  id = "sgr-0b379f87680a2a333"
}
# End

-----------------


[root@ip-172-30-2-182 sg2]#cd sg3
[root@ip-172-30-2-182 sg2]#terragrunt plan --generate-config-out=out.tf

You should see Plan: 6 to import, 0 to add, 0 to change, 0 to destroy.

This seems correct. When we run plan, we see all the 6 resources to be imported.

--generate-config-out=out.tf  will create a out.tf file in the terraform cache folder (example: .terragrunt-cache folder)

out.tf file has the current resource attributes fetched from AWS and ready for import.

Note:  if you decide to modify the above sg.tf file, make sure to delete out.tf file and re-run the plan command.


perform apply.
[root@ip-172-30-2-182 sg2]#terragrunt plan

Apply should succeed.  Apply complete! Resources: 6 imported, 0 added, 0 changed, 0 destroyed.

Verify import using terragrunt state list.
#terragrunt state list
aws_security_group.this_1
aws_security_group.this_2
aws_vpc_security_group_egress_rule.egress_1
aws_vpc_security_group_egress_rule.egress_2
aws_vpc_security_group_ingress_rule.ingress_1
aws_vpc_security_group_ingress_rule.ingress_2

#terragrunt state show aws_security_group.this_1
partial data
resource "aws_security_group" "this_1" {
    arn         = "arn:aws:ec2:us-east-1:11111:security-group/sg-00740ac44c8417888"
    description = "lg-sg"
    egress      = [
        {
            cidr_blocks      = [
                "0.0.0.0/0",
            ]
            description      = null
            from_port        = 0
            ipv6_cidr_blocks = []
            prefix_list_ids  = []
            protocol         = "-1"
            security_groups  = []
            self             = false
            to_port          = 0
        },
    ]
    id          = "sg-00740ac44c8417888"
    ingress     = [
        {
            cidr_blocks      = [
                "0.0.0.0/0",
            ]
            description      = null
            from_port        = 80
            ipv6_cidr_blocks = []
            prefix_list_ids  = []
            protocol         = "tcp"


Story 1 is completed.  Import is successful and resources are now under terragrunt control.


**STORY 2**
**Updating CIDR egress and ingress from terragrunt.**





