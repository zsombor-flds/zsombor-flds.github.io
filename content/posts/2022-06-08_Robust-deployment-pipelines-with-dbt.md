---
title: Robust deployment pipelines with dbt
description: "An in-depth guide to best practices for stable analytical warehouse releases using dbt projects, including continuous integration, blue-green deployment, and tools like sqlfluff and Spectacles."
date: '2022-06-08T10:14:18.998Z'
tags: ["DevOps", "snowflake", "dbt"]
type: post
weight: 25
showTableOfContents: true
---

I think a robust Release process is an unavoidable accessory for a stable Analytical warehouse. With a stable deployment cycle, we can increase the reliability of the platform. In the following next paragraphs, I will give an insight into the best practices we use in our dbt projects.

Luckily, we have the option of using the out-of-the-box solutions here, but we can extend the default option in several ways.

In my opinion, a Deployment process starts with a good Pull Request. You can read about PR best practices [here](https://medium.com/@meshantha/guide-in-making-your-perfect-pull-request-d6f757f73276), the PR template we use can be found [here](https://docs.getdbt.com/blog/analytics-pull-request-template).

### **Continuous integration with dbt Cloud**

Continuous integration (CI) is a software engineering practice in which changes to the code base are tested immediately.

The standard solution provided by dbt allows each PR to create a temp schema in which to run a dbt run + test command, to verify that nothing has break during the development.

![](/images/0__42URRRJdietzTkCc.jpg)

How to setup: [link](https://docs.getdbt.com/docs/dbt-cloud/using-dbt-cloud/cloud-enabling-continuous-integration)

With the [slim CI](https://discourse.getdbt.com/t/how-we-sped-up-our-ci-runs-by-10x-using-slim-ci/2603) option, we have the ability to run only the modified models + their downstream models — reducing the CI job run time.

```
dbt build --select state:modified+
```

Note that while it’s quicker to set up a CI pipeline with dbt Cloud, there are several ways to implement this with dbt core as well.

#### Zero Copy Clone CI on Snowflake

Another great way to further optimize the CI pipelines is to leverage the [Zero Copy Clone](https://docs.snowflake.com/en/sql-reference/sql/create-clone.html) feature in Snowflake.

With this approach Instead of creating temp schema for every CI job you create a copy of the PROD database and run the test on the clone environment to ensure that cross-references is up to date in a deferred run. You can find an awesome detailed guide on how to setup a ZCC CI job [here](https://medium.com/airtribe/test-sql-pipelines-against-production-clones-using-dbt-and-snowflake-2f8293722dd4).

### Blue Green Deployment

This deployment method is inherited from software development.

> By definition:  
> “Blue green deployment is an application release model that gradually transfers user traffic from a previous version of an app or microservice to a nearly identical new release — both of which are running in production.”

![](/images/0__JVyrgoij3JnzdiHs.jpg)

With this deployment method, we have two identical environments(PROD, PREPROD). The production job always is running in the PREPROD environment then if the job is finished with a success (e.g all tests run successfully, every table has been updated) we swap the two environments. -> this approach radically reduces downtime.

You can read about how to set this up here: [link](https://discourse.getdbt.com/t/performing-a-blue-green-deploy-of-your-dbt-project-on-snowflake/1349)

(Important Note: The community prefers the word combination STAGE and PROD in practice. However, I think this can cause misunderstandings, as the first layer of the database is also called a staging layer, not to mention that Snowflake’s external tables also have Stages. )

### More best practices

My two favorite additions are [sqlfluff](https://github.com/sqlfluff/sqlfluff) and [pre-commit-dbt](https://pythonlang.dev/repo/offbi-pre-commit-dbt/) . Both tools can easily be used as a Github Action or as a git ho.

#### sqlfuff

[Sqlfluff](https://github.com/sqlfluff/sqlfluff) is the go-to linter for dbt projects, it enforces stylistic rules on your code. These rules only change the way the code is formatted, not the business logic behind it. This will ensure that the project developers follow the code style guide, no matter if they are [leading or trailing comma](https://mode.com/blog/should-sql-queries-use-trailing-or-leading-commas/) persons.

![](/images/1____IZvE1EpsBqPGzA9W0GV6A.png)

#### pre-commit-dbt

With [pre-commit-db](https://github.com/offbi/pre-commit-dbt)t, you can ensure that your project stays in order as the number of models increases.

For example, you can automatically test whether the columns in the models have a [description](https://github.com/offbi/pre-commit-dbt/blob/main/HOOKS.md#check-model-columns-have-desc). This speeds up Code Reviews since you can test a lot of formal requirements in an automated fashion.

#### **Spectacles** (for Looker users)

The previous points are natively available with dbt (at most with a bit of hacking). Spectacles, however, takes advantage of the [data OS](https://benn.substack.com/p/the-data-os?s=r) aspect of the ecosystem.

With this tool, we have the opportunity to do testing not only at the database level but also in the Looker dashboards sitting on the top of the project.

![](/images/0__K__eGH6wykgDSUAK8.jpg)

Spectacles use the Looker API to test whether changes to the database have broken something on the frontend side, ensuring that your Looker Explores actually work.

More on Spectacles here: [link](https://www.spectacles.dev/)