# terraform-best-practices

Terraform Best Practices for AWS users.

* [Run terraform command with var-file](#run-terraform-command-with-var-file)
* [Retrieves state meta data from a remote backend](#retrieves-state-meta-data-from-a-remote-backend)
* [Use share modules](#use-share-modules)
* [Isolate environment](#isolate-environment)
* [Use terraform import to include as more resources you can](#use-terraform-import-to-include-as-more-resources-you-can)
* [Avoid hard code the resources](#avoid-hard-code-the-resources)
* [Format terraform codes](#format-terraform-codes)
* [Enable version control on terraform state files bucket](#enable-version-control-on-terraform-state-files-bucket)
* [Generate README for each module about input and output variables](#generate-readme-for-each-module-about-input-and-output-variables)
* [update terraform version](#update-terraform-version)
* [Run terraform from docker container](#run-terraform-from-docker-container)

## Run terraform command with var-file

```
$ cat config/dev.tfvars

name = "dev-stack"
s3_terraform_bucket = "dev-stack-terraform"
tag_team_name = "hello-world"
 
$ terraform plan -var-file=config/dev.tfvars 
```

With `var-file`, you can easly control which environment you will work on and needn't follow with a lot of var key-value pairs ( `-var foo=bar` )

## Retrieves state meta data from a remote backend

Normally we have several layers to manage terraform resources. After you create the base resources, such as vpc, security group, subnets, nat gateway. You should refer the states directly from vpc layer.

```
data "terraform_remote_state" "stack" {
  backend = "s3"
  config{
    bucket = "${var.s3_terraform_bucket}"
    key = "${var.stack_name}/terraform.tfstate"
    region="${var.aws_region}"
  }
}
 
# Retrieves the vpc_id and subnet_ids directly from remote backend state files.
resource "aws_xx_xxxx" "main" {
  # ...
  subnet_ids = "${split(",", data.terraform_remote_state.stack.data_subnets)}"
  vpc_id     = "${data.terraform_remote_state.vpc.vpc_id}"
}
```

## Use share modules

Manage terraform resource with shared modules, this will save a lot of coding time. 

### NOTES:

terraform modules don't support `count` parameter currently. You can follow up this ticket for updates: https://github.com/hashicorp/terraform/issues/953

[terraform module usage](https://www.terraform.io/docs/modules/usage.html)

[Terraform community modules](https://github.com/terraform-community-modules)

## Isolate environment

Someone create a security group and share it to all non-prod (dev/qa) environments. Don't do that, create resources with different environment name for each environment.


## Use terraform import to include as more resources you can

Sometimes developers created some resources directly to rush. You need to mark these resource and use terraform import to include them in codes. 

## Avoid hard code the resources

A sample:
```
account_number=“123456789012"
```

The aws account id or account alias can be input directly via data sources.

```
# The attribute `${data.aws_caller_identity.current.account_id}` will be current account number. 
data "aws_caller_identity" "current" {}

# The attribue `${data.aws_iam_account_alias.current.account_alias}` will be current account alias
# Tips: you can easly use this attribue to create terraform bucket with environment, project name, etc.
data "aws_iam_account_alias" "current" {}
```

Refer: [terraform data sources](https://www.terraform.io/docs/providers/aws/)

## Format terraform codes

Always run `terraform fmt` to format terraform configuration files and make them neatly.

## Enable version control on terraform state files bucket

Always set backend to s3 and enable version control on this bucket. 

If you'd like to manage this bucket as well, recommend to use this repostory [terraform-state-bucket](https://github.com/BWITS/terraform-state-bucket) to create the bucket and replica to other regions automatically. 

## Generate README for each module about input and output variables

Show the command running on mac:
```
$ brew install terraform-docs
$ cd terraform/modules/vpc
$ terraform-docs md . > README.md
```

For details on how to run terraform-docs, check this repository: https://github.com/segmentio/terraform-docs

## update terraform version

Hashicorp doesn't have a good qa/build/release process for their software and does not follow semantic versioning rules.

For example, `terraform init` isn't compatible between 0.9 and 0.8. Now they are going to split providers and use "init" to install providers as plugin in coming version 0.10

So recommend to keep updating to latest terraform version

## Run terraform from docker container

Terraform releases official docker containers that you can easly control which version you can run.

Recommend to run terraform docker container, when you set your build job in CI/CD pipeline.

```
TERRAFORM_IMAGE=hashicorp/terraform:0.9.8
TERRAFORM_CMD="docker run -ti --rm -w /app -v ${HOME}/.aws:/root/.aws -v ${HOME}/.ssh:/root/.ssh -v `pwd`:/app $TERRAFORM_IMAGE"
```

