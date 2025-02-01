---
title: How to run Spark 3 Glue jobs locally with docker?
description: "Discover how to locally develop AWS Glue jobs using a dockerized Spark 3.1 engine, bypassing the need for a constantly running AWS developer endpoint"
date: '2021-10-12T16:13:53.062Z'
tags: ["docker", "spark", "python"]
type: post
weight: 25
showTableOfContents: true
---

_The original article can be found on_ [_Hiflylabs’ blog_](https://blog.hiflylabs.hu/en/2021/10/12/spark3gluejobs/)

AWS Glue development requires that a developer endpoint should be running at all times. (In fact, technically it only has to run when the jobs are to be launched; however stopping the endpoint is not possible, and killing and re-creating it requires config changes which is a major hassle.)

For smaller teams, in small or hobby projects it makes a lot of sense to develop and run Glue jobs locally, independently of AWS. This is possible with dockerized Spark — but AWS provides only limited support.

Although Spark 3 came out early June 2020, unfortunately AWS currently (as of October 2021) only provides a docker image with Spark 2.4. Fortunately Spark 3.1 engine is available in the cloud via the AWS console.

It is beyond the scope of this post to contemplate on whether it’s worth switching from 2 to 3, I just want to show you how you can run Glue jobs locally with the Spark 3.1 engine.

Note that this solution is based on the [alruen](https://github.com/alrouen/local-aws-glue-v3-zeppelin) attempt, but that didn’t quite work for our purpose. (However, I would like to give a thumbs up from here as well!)

Without further ado, you can try our pre-build image from [**_here_**](https://hub.docker.com/repository/docker/hiflylabs/local-aws-glue-v3-zeppelin) or [**_here_**](https://github.com/Hiflylabs/aws-glue-spark3-docker) you will find everything you need to know for local build or customization.

Beyond docker, you need the AWS command line interface ([AWS cli](https://aws.amazon.com/cli/)). After installing it you must authenticate with the aws-configure command.

You can then start the container with the following command:
```bash
docker run -it --rm --name local-glue -p 8080:8080 -p 9001:9001 -v $PWD/logs:/logs -v $PWD/notebook:/notebook -e ZEPPELIN\_LOG\_DIR=’/logs’ -e ZEPPELIN\_NOTEBOOK\_DIR=’/notebook’ -v ~/.aws:/root/.aws:ro <todo-hf-docker-image-name>
```

Note that the ~/.aws:/root/ path represents the default location of the aws configuration files (which were created by the aws-configure command).

Once the container has started, you can start coding in the Zeppelin notebook, which you can access at [http://127.0.0.1:9001,](http://127.0.0.1:9001,) or you can attach the running container from VS code. Although it is not part of this tutorial, you can read more about this [here](https://code.visualstudio.com/docs/remote/attach-container).

![](/images/1__P256v5kNryDrZeeXHuBtCQ.png)

If all went well, you can now successfully develop AWS glue jobs locally on your own machine with Spark version 3; you don’t need either the AWS console nor a developer endpoint.