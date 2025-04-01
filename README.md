# import_sg_terragrunt
This is a demo repository for importing and managing the AWS security groups.
#Purpose
Lot of security groups may have been created using Clickops. it makes more sense to stop using clickops and bring these resources under Terraform control. Terragrunt is wrapper around Terraform and provides many benefits (state management and keeping code DRY primarily.)

#Scenario below:
We will import 2 security groups and its connected egress/ingress rules to Terragrunt control. Once imported, we will also update the rules in the existing security groups, modify other attributes.  After that, we will also add new security groups and rules using Terragrunt. No more clickops.
If someone created more security groups again, use the same process to import back to Terragrunt.

Step 1:
Use an desktop or an ec2 instance where Terragrunt/terraform is installed.  I use an AWS Amazon linux instance.  It has permission to manage resources from my personal AWS account using AWS IAM role. This setup detail is not covered here.

[root@ip-172-30-2-182 sg3]# terragrunt --version
terragrunt version v0.75.4

Terragrunt uses one directory and only one terragrunt.hcl to mange the resources. Actual public module will be stored in the ./modules folder.
In this case, we have sg3/terragrunt.hcl

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



cat ../../modules/general_import/sg.tf
Note: I create a folder called "general_import" and store the below content in the file called sg.tf.  it can be any name ending with .tf.
List out all the security groups and rules (egress/ingress) to be imported as shown.
Reason we are using rules separate from security group is because "rules" are considered separate resources in the Terraform documentation. (AWS VPC rules are different than AWS VPC security groups. each has its own published module or resource block.
Some resources are published as modules and some as just resources.  
Hint: go to https://registry.terraform.io/providers/hashicorp/aws/latest/docs  search for "aws security group" and you will get https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws/latest
click on source code link and open main.tf file and look for the resource name which is "resource "aws_security_group" "this"   
use "aws_security_group".  Subgroup "this" can be anything but must be unique within a tf file. So we use this_1

Similary to find the CIDR rule module or resource go to https://registry.terraform.io/providers/hashicorp/aws/latest/doc search for 
 
  
In this case, aws security group has a module at
rules have a resource at:
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


