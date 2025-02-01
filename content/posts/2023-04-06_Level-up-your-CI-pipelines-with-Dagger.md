---
title: Level up your CI pipelines with Dagger.
description: How Dagger Makes Life Easier for Analytics Engineers.
date: '2023-04-06T10:51:28.164Z'
tags: ["CI/CD", "DevOps", "docker", "dagger"]
type: post
weight: 25
showTableOfContents: true
---

![](/images/1__zI2snQGcamZMusTntQD1dg.png)

A typical Analytics Engineer’s development workflow consists of many tools.   
We use sqlfluff to lint the models, git hooks to raise issues automatically, Spectacles for testing Looker, and the Datafold CLI to compare datasets. This list varies based on the different business needs/environments.

Often you want to execute these processes:

*   locally before a commit/PR
*   automatically as part of some CI automation

Tools tend to use different interfaces for invocation. You can use some of them as git hooks and others with fancy CLIs, but managing/setup all this can be time-consuming and challenging for less tech-savvy Analytics Engineers.

Previously, I used to wrap most of these as GitHub Actions. If you’re lucky, you can find GitHub Actions provided by the vendor or the community that you can plug and play, but in many cases, you need to create your workflow. This setup works well, but not all CI tools have the same thrilling community as [GitH](https://github.blog/2019-11-06-a-thousand-community-powered-workflows-using-github-actions/)ub.

Once the action was properly set up, I used a tool called [act](https://github.com/nektos/act) to execute the CI pipelines locally.

Although I liked this setup because it enabled us to evaluate changes faster, with this approach, it was still painful to reproduce these actions in different CI runners (e.g., from GitHub Actions to Gitlab-CI).

![](/images/0__bgVJX8QUXvkPCyU7.jpg)

In the following paragraphs, I will show you how easy it is to solve these problems using [Dagger](https://github.com/dagger/dagger).

### **So, what on earth is Dagger, and why should you care?**

[Dagger](https://dagger.io/) is an open-source programmable [CI/CD](https://en.wikipedia.org/wiki/CI/CD) engine created by [Solomon Hykes](https://www.linkedin.com/in/solomonhykes/), the founder of [Docker](https://www.docker.com/). Dagger makes it easy to develop portable CI pipelines in your favorite programming language that executes entirely on standard OCI containers..

![](/images/0__y0H2F5D0OngJSp84.jpg)

> ‘_Dagger does not replace your CI: it improves it by adding a portable development layer on top of it.’_  
> src: [https://docs.dagger.io/1220/vs/](https://docs.dagger.io/1220/vs/)

### CI/CD as Code

Unlike conventional CI/CD tools, Dagger lets you write a pipeline as code instead of writing proprietary YAML, you can write your CI/CD code in CUE, Go, Python, or NodeJS. This approach makes it easy to create dynamic pipelines and enables you to test your CI/CD just like any other project element.

### Run Anywhere

You can test and debug instantly on your local machine. No need to push your changes to trigger the CI pipeline can run it anytime in your own isolated environment. This enables you to get instant feedback on the impact of your changes.

Dagger executes your pipelines entirely as [standard OCI containers](https://opencontainers.org/). Therefore, it is compatible with most CI/CD runtime environments, including:

*   GitHub Actions
*   GitLab-CI
*   Travis-CI

This approach has the benefit of enabling you to execute the same CI process everywhere.

![](/images/0__njBFj8JfMFbS9EvW.jpg)

### Performance

Caching across pipeline runs is one of Dagger’s most potent but often overlooked power.

You can designate one or more directories as cache volumes in your pipeline, and its content will be persistent across runs. This makes it possible to reuse the cache’s contents at each pipeline run, which speeds up pipeline operations.

### Demo

To demonstrate how easy it is to set up a CI process, let’s write our first pipeline with Dagger.

Our dummy pipeline will:

*   initialize the Dagger client
*   mount the dummy projects dir to the container
*   install some Python dependencies
*   run the linter by executing the sqlfmt command

![](/images/0__it3qp1sK6FLd25W1.jpg)

Simple as that, we were able to create a universal CI pipeline that can be executed on different CI runners.

You can execute the example locally:

![](/images/0__xJxleOMYPrCxGVxY.jpg)

Or wrap the same pipeline as Github Actions just like this:

![](/images/0__XOOULD8WtA__lfjtA.jpg)

This demonstrates how Dagger makes it easy to create custom pipelines that fit together like Lego pieces. Now, my CI setup is primarily built around Dagger for the build, testing, publish pipelines, and with this approach, I can run my CI anywhere I can run Docker containers.

In conclusion, Dagger is a powerful tool that enables Analytics Engineers to create portable CI pipelines in their favorite programming language and execute them in any Docker-compatible runtime environment.