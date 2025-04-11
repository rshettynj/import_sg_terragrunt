
# IAM policies and role creation using Terragrunt and Terraform.  

**To create an IAM role using terragrunt **
```
[root@ip-172-30-2-182 iam-assumable-role]# cat terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
#below source is coming from terraform.io which is placed in the below folder called iam-assumable-role
  source = "../../modules/terraform-aws-iam/modules/iam-assumable-role"
}

inputs = {
  create_role = true
  role_name = "my-terraform-test-role"
  role_path = "/"
  role_description = "my terraform test role"
  max_session_duration = 3600
  create_custom_role_trust_policy = true
  trusted_role_arns = [ "arn:aws:iam::55999490xxxx:root" ]
  tags = {
    name = "testing"
    purpose = 2324
  }
}
```

**To create an IAM policy using terragrunt **
```
[root@ip-172-30-2-182 iam-policy]# cat terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../modules/terraform-aws-iam/modules/iam-policy"
}

inputs = {
  create_policy = true
  name = "my-terraform-test-policy"
  path = "/"
  description = "testing terraform for iam policy"
  policy = <<EOT
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": [
        "s3:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]

}
EOT
  tags = {
    name = "testing"
    purpose = 2324
  }
}
```

** To create multiple AWS IAM policies using terragrunt (one terragrunt.hcl to manage multiple IAM policies. **
```
[root@ip-172-30-2-182 iam-policy]# cat terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../modules/terraform-aws-iam/modules/iam-policy"
}

inputs = {

#We are creating a map called _myListofObjects below and wrapping the above module call.

  _myListofObjects = [
{
  create_policy = true
  name = "my-terraform-test-policy"
  path = "/"
  description = "testing terraform for iam policy"
  policy = <<EOT1
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": [
        "s3:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]

}
EOT1
  tags = {
    name = "testing 2"
    purpose = "demo iam 2"
}

},

#Next list item
{
  create_policy = true
  name = "my-terraform-test-policy-2"
  path = "/"
  description = "testing terraform for iam policy-2"
  policy = <<EOT2
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": [
        "s3:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOT2
  tags = {
    name = "testing 2"
    purpose = "demo iam"
  }
}
#list item End
,
#Next list item
{
  create_policy = true
  name = "my-terraform-test-policy-3"
  path = "/"
  description = "testing terraform for iam policy-3a"
  policy = <<EOT2
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": [
        "ec2:*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOT2
  tags = {
    name = "testing 3a"
    purpose = "demo iam"
  }
}
#list item End


]
}
```

cat main.tf (modified !!!)
```
resource "aws_iam_policy" "policy" {
  #count = var.create_policy ? 1 : 0

/*
  name        = var.name
  name_prefix = var.name_prefix
  path        = var.path
  description = var.description
  policy = var.policy
  tags = var.tags
*/
for_each = { for index, inst in var._myListofObjects : index => inst }
  name        = each.value.name
  #name_prefix = each.value.name_prefix
  path        = each.value.path
  description = each.value.description
  policy      = each.value.policy
  #tags        = each.value.tags
}
```

```
cat variables.tf
variable "_myListofObjects" {
  description = "List of IAM policies to create"
  type = list(object({
    create_policy = bool
    name = string
    path = string
    description = string
    policy = string
  }))
}
```

What is possible now?

With for_each added in the main.tf file and terragrunt.hcl is now sending "list of objects" (map of objects), we can manage (create, delete selectively and update selectively) any of the IAM policies listed in the 
terragrunt.hcl file. Which means, one file is used to manage as many IAM policies (or any resources.).

After the terragrunt apply, resources will be created. Double check the state list command to confirm as shown below.

```
[root@ip-172-30-2-182 iam-policy]# terragrunt state list
aws_iam_policy.policy["0"]
aws_iam_policy.policy["1"]
aws_iam_policy.policy["2"]
```

```
[root@ip-172-30-2-182 iam-policy]# terragrunt state show 'aws_iam_policy.policy["2"]'
resource "aws_iam_policy" "policy" {
    arn              = "arn:aws:iam::559994907943:policy/my-terraform-test-policy-3"
    attachment_count = 0
    description      = "testing terraform for iam policy-3a"
    id               = "arn:aws:iam::559994907943:policy/my-terraform-test-policy-3"
    name             = "my-terraform-test-policy-3"
    name_prefix      = null
    path             = "/"
    policy           = jsonencode(
        {
            Statement = [
                {
                    Action   = [
                        "s3:ListAllMyBuckets",
                    ]
                    Effect   = "Allow"
                    Resource = "*"
                },
                {
                    Action   = [
                        "ec2:*",
                    ]
                    Effect   = "Allow"
                    Resource = "*"
                },
            ]
            Version   = "2012-10-17"
        }
    )
    policy_id        = "ANPAYEYSMFUTWMWACAGIL"
    tags             = {}
    tags_all         = {}
}
```

below command will only delete the third IAM resource.
[root@ip-172-30-2-182 iam-policy]# terragrunt destroy -target='aws_iam_policy.policy["2"]'
Note:  Ensure that resource mentioned here is not dependent on some other resource, if so those will be removed as well.

Caution:  I am not sure yet what happens as resource is deleted and numbers come down. this may be an issue with array counter. Need to test.


Next
Use the Policy ARN in the Role and attach the policy/policies.
