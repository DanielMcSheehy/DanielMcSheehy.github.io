---
layout: post
title: Let customers install your integrations using Terraform
description: How to let customers manage permissions and third party integrations with Terraform
summary: 
tags: [terraform]
---
When I was working at effx, customers had to go through a lengthy process for installing many of our integrations. 

For AWS, they had to create a new custom policy that granted us read only permissions that we needed to pull metadata from their account. Then they had to grant a role with that policy based on our id. 

The whole process took 5-10 minutes and frequently resulted in misconfiguration. 

Is there a better way?

Terraform is a tool that allows "infrustructure as code". Whatever resources/permissions they 
specify, will be present in their cloud service provider. 

At effx, I wrote a 
[library](https://github.com/effxhq/terraform-aws-iam)
that allows customers to instantly grant us read only access. 
It also is transparent about what access we require, and easy to disable. 

below is all they would have to add to their Terraform config to allow access to 
ecs. 

```terraform 
module "effx_aws_integration" {
  source  = "git::github.com/effxhq/terraform-aws-iam.git?ref=main"
  enable_ecs = true // defaults to true
}
```

How it works under the hood

The below resource creates a json aws policy that looks familar.
```hcl
resource "aws_iam_policy" "effx_aws_ecs_integration" {
  count = var.enable_ecs ? 1 : 0
  name  = "effx-AWS-ECS-Integration-Policy"

  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Action" : [
          
          "ecs:Describe*",
          "ecs:List*",
          "health:DescribeEvents",
          "health:DescribeEventDetails",
          "health:DescribeAffectedEntities",
        ],
        "Effect" : "Allow",
        "Resource" : "*"
      }
    ]
  })
}
```

And then this creates the role based on the above policy 
```hcl 
resource "aws_iam_role" "effx_aws_ecs_integration_role" {
  name                = "effx-AWS-ECS-Integration-Role"
  assume_role_policy  = data.aws_iam_policy_document.instance_assume_role_policy.json
  managed_policy_arns = [effx_aws_ecs.iam_policy_arn, effx_aws_lambda.iam_policy_arn]

  tags = var.resource_tags
}
```



