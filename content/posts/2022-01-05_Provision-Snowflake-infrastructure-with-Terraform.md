---
title: Provision Snowflake infrastructure with Terraform
description: "An overview of automating Snowflake component provisioning using Terraform in dbt projects, detailing the benefits of Infrastructure as Code and offering step-by-step guidance on setting up dynamic resource generation with code examples"
date: '2022-01-05T12:59:02.547Z'
tags: ["snowflake", "devops", "terraform"]
type: post
weight: 25
showTableOfContents: true
---

_The original article can be found on_ [_Hiflylabs’ blog_](https://blog.hiflylabs.hu/en/2022/01/05/snowflake/)

In this post, I want to show you how we have automated the provisioning of Snowflake components with Terraform in our dbt projects.

Terraform is an open-source, **I**nfrastructure **a**s a **C**ode (IaC) tool that is used to manage the cloud services with a declarative, version controlled configuration format. It is cloud-agnostic so it works on AWS, GCP, Azure and many other cloud providers. Its benefits include that you can write maintainable, reusable, modular pieces of code. So it fits really well with the mindset that makes so many of us love using dbt.

In the upcoming paragraphs I will focus on the [Snowflake provider](https://registry.terraform.io/providers/chanzuckerberg/snowflake/latest).

First you need to install the Terraform CLI. To do this, follow the instructions here: [install-cli](https://learn.hashicorp.com/tutorials/terraform/install-cli)

Note that this varies from OS to OS, on MacOS, for example, I had to run these commands:

```bash
$ brew tap hashicorp/tap  
$ brew install hashicorp/tap/terraform
```

After a successful installation, we need to initialize the Terraform project.  
To do this we need a new directory, then create the main.tf file with the following content:
```
terraform {  
  required\_providers {  
    snowflake = {  
      source  = "chanzuckerberg/snowflake"  
      version = "0.25.28"  
    }  
  }  
}
```

From there run the following command:

```bash
$ terraform init
```

![](/images/1__JRlYkuzOHbl8UNnXVIBUiQ.gif)

The project will consist of the following files:

*   main.tf : call modules, locals and data-sources to create all resources
*   variable.tf : contains declarations of variables used in main.tf
*   secret.tfvars : sensitive variables, such as passwords

Without the use of Terraform on projects, Snowflake dependencies need to be created manually. For example, in a new project, one thing that you definitely need in a Snowflake tenant is at least one warehouse:

It looks like this in pure SQL:

```
CREATE WAREHOUSE TRANSFORMING WITH   
 WAREHOUSE\_SIZE = ‘XSMALL’   
 WAREHOUSE\_TYPE = ‘STANDARD’   
 AUTO\_SUSPEND = 600   
 AUTO\_RESUME = TRUE;
```

There is nothing fundamentally wrong with this approach, but there is no clear way to manage this kind of code in your dbt project. You can save it to a new repo or to one of the corners of your project, but there is no trivial solution to manage these external dependencies, and to make it parametrizable, reusable. Managing these external resources can cause technical debt.

The first step is to set up the required providers. This is necessary because the technical user on whose behalf we run the commands needs different roles for different operations.(This user must be at least SYS, SECURITY admin within Snowflake.)

Add the following to the contents of main.tf:

```
provider "snowflake" {  
username = <snowflake-username>  
password = <snowflake-password>  
account  = <snowflake-account-name>  
region   = <snowflake-region-name>  
alias    = "sys\_admin"  
role     = "SYSADMIN"  
}
```

(of course, sensitive information will not be stored in main.tf later)

To create a Snowflake warehouse in Terraform is as simple as this:

```
resource snowflake\_warehouse w {  
  name           = "test"  
  comment        = "foo"  
  warehouse\_size = "small"  
}
```

It’s looks something like this:

![](/images/1__NuzPtvzokE5VgdSsbW9NLg.gif)

We successfully provisioned a Snowflake warehouse from Terraform.

Now we are going to make the process more sophisticated. We will derive each value into a variable, for example, if you need more than one warehouse (which you probably do).

With the following DRY approach, it’s easy to automate the creation of warehouses in a configurable and reusable way.   
You can simply add something this to your variable.tf:

```
variable "warehouses" {  
  type = map(object({  
    name = string,  
    size = string  
  }))  
  default = {  
    "test01" = {  
      name = "TEST\_WAREHOUSE",  
      size = "xsmall"  
    }  
    "test02" = {  
      name = "TEST\_WAREHOUSE\_2",  
      size = "xsmall"  
    }  
}
```

main.tf

```
resource "snowflake\_warehouse" "warehouses" {  
  for\_each            = var.warehouses  
  provider            = snowflake.sys\_admin  
  name                = upper(each.value.name)  
  warehouse\_size      = each.value.size  
  auto\_suspend        = 60  
  initially\_suspended = true  
}
```

So, as described above, with Terraform we can dynamically generate the necessary resources. I think this is a good way to demonstrate the power of Terraform.

![](/images/1__l1pazYOdd79FsMZ97lfzCg.gif)

Here is high level view of our final bootstrapping project:   
This will create the following objects:

*   create warehouses
*   create databases
*   create roles
*   create dbt user
*   create schemas
*   grant roles

![](/images/1__vKorH7GLvGc4b7WhLSfw7g.png)

You can find the source code here: [link](https://github.com/Hiflylabs/snowflake-terraform-boostrap)

In this post, I tried to show you how to provision Snowflake infrastructure with Terraform.   
Development of the dbt Cloud Terraform provider started 3 months ago (Oct 3, 2021), which will allow us to put dbt Cloud resources under version control in the future as well.
