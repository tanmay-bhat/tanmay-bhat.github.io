---
layout: post
title: 'How to Move a Terraform Resource into a Module Using Moved Block'
date: 2023-11-04
tags: ["terraform", "aws"]
---
 Starting from v1.1, Terraform provides a powerful feature known as the `moved` block. This feature allows you to reorganize your Terraform configuration without causing Terraform to perceive the refactor as a deletion and creation of resources. In this article, we will walk through a few examples of Terraform refactoring using the `moved` block.

## Prerequisites

- Terraform ( >=1.1)

## Move a Resource Into Module

First, we will create a sample S3 bucket to reference as a standalone resource. In your Terraform configuration directory, create a new `./lab-demo/s3.tf` file.

```terraform
resource "aws_s3_bucket" "moved_demo" {
  bucket = "moved-demo"
  tags = {
    env = "lab"
  }
}
```

Then, initialize Terraform and apply the configuration, which will create the S3 bucket. If we list

```bash
terraform state list
aws_s3_bucket.moved_demo
```

Now, let's create a module directory `./modules/aws/s3` and move the `./lab-demo/s3.tf` into `./modules/aws/`. The directory structure should look like this:

```bash
.
├── lab-demo
│   ├── main.tf
│   ├── s3.tf # Moved to ./modules/aws/s3/s3.tf
│   ├── modules
│       └── aws
│           └── s3
│               └── s3.tf
```

Since we have the resource in the module, we need to update the reference in the `./lab-demo/main.tf` file.

```terraform
module "s3" {
  source = "./modules/aws/s3"
}
```

If we run `terraform init && terraform plan` now, Terraform will propose to destroy and recreate the same S3 bucket. This is because Terraform thinks that the object reference does not exist anymore. 

```terraform
  # aws_s3_bucket.moved_demo will be destroyed
  # (because aws_s3_bucket.moved_demo is not in configuration)
  ...
   # module.s3.aws_s3_bucket.moved_demo will be created
```

To fix this, we need to add a `moved` block to the `./lab-demo/main.tf` file.

```terraform
module "s3" {
  source = "./modules/aws/s3"
}

moved {
  from = aws_s3_bucket.moved_demo
  to   = module.s3.aws_s3_bucket.moved_demo
}
```

If you do the `terraform plan` again, Terraform will recognize the move and not propose to destroy and recreate the resource.

```terraform
  # aws_s3_bucket.moved_demo has moved to module.s3.aws_s3_bucket.moved_demo
    resource "aws_s3_bucket" "moved_demo" {
        id                          = "moved-demo"
        tags                        = {
            "env" = "lab"
        }
    }
Plan: 0 to add, 0 to change, 0 to destroy.
```

Once you apply the configuration, you can remove the `moved` block from the `./lab-demo/main.tf` file.

## Moving Resource Between Modules

The `moved` block can also be used to move resources between modules. Let's create a new module `./modules/aws/another_module` and move the `./modules/aws/s3/s3.tf` into `./modules/aws/another_module`. The directory structure should look like this:

```bash
.
├── lab-demo
│   ├── main.tf
│   ├── modules
│       └── aws
│           ├── s3
│           └── another_module
│               └── s3.tf # Moved from ./modules/aws/s3/s3.tf
```

Since we have the resource in the module, we need to update the reference in the `./lab-demo/main.tf` file.

```terraform
module "s3" {
  source = "./modules/aws/another_module"
}
```

In order to move the resource from `s3` module `another_module` , we can use the same `moved` block to the `./lab-demo/main.tf` file.

```terraform
module "s3" {
  source = "./modules/aws/s3"
}

moved {
  from = module.s3.aws_s3_bucket.moved_demo
  to   = module.another_module.aws_s3_bucket.moved_demo
}
```

And the `terraform plan` will show the resource move.

```terraform
  # module.s3.aws_s3_bucket.moved_demo has moved to module.another_module.aws_s3_bucket.moved_demo
    resource "aws_s3_bucket" "b" {
        id                          = "moved-demo"
        tags                        = {
            "env" = "lab"
        }
    }
Plan: 0 to add, 0 to change, 0 to destroy.
```

### Renaming a Module

Similarly, we can also rename the module using the `moved` block. Let's rename the `another_module` to `s3` and move the resource back to the `s3` module. Below is an example `main.tf` file.

```terraform
## Before: main.tf

module "s3" {
  source = "./modules/aws/s3"
}
```

```terraform
## After: main.tf

module "s3_renamed_module" {
  source = "./modules/aws/s3"
}

moved {
  from = module.s3
  to   = module.s3_renamed_module
}
```

In this case, all the resources in the `s3` module will be moved to the `s3_renamed_module` module.

---
### Resources :
https://developer.hashicorp.com/terraform/language/v1.1.x/modules/develop/refactoring