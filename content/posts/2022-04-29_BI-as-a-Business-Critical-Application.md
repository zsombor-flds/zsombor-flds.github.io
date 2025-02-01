---
title: BI as a Business-Critical Application
description: >-
  As more companies begin to adapt their Analytics platform with dbt, one demand
  from enterprise users is becoming more apparent: Data…
date: '2022-04-29T11:23:34.459Z'
tags: ["dbt"]
type: post
weight: 25
showTableOfContents: true
---

As more companies begin to adapt their Analytics platform with dbt, one demand from enterprise users is becoming more apparent: Data Reliability.

Companies are investing more and more money to exploit the available data, and the exposure to data quality/availability is growing exponentially.

BI teams are no longer isolated within the organizations but contribute to the business continuity at the operational level. So we have to accept that modern BI platforms have become business-critical applications. Analytical DWHs are no longer used only by business analysts for reporting, but the data is circulated back to the core business systems.

Fully automated data-based applications amplify the competitiveness of companies, so bad data or faulty KPIs can have extremely detrimental consequences.

In the next few posts, I will try to summarize the building blocks that can make a data platform more robust.

We need to be aware that dealing with data quality issues has always been present in BI, but today we have more mainstream options to deal with these challenges than 5–10 years ago.

To put it simply, the motivation is still the same, we should be aware of the data errors before they hit production, and surely before the end-users recognize them. Better have an automated warning and a paused data load about a missing column than an email from the CEO telling us that the “X system” is down “again”.

(Note: I have consciously omitted a very important dimension from my thought process.   
In the evolution, BI platforms have become complex and diverse.  
Now we do not just have multiple variant input data sources with different characteristics, and large amounts of data volumes, but also have to deal with infinitely more data consumers of the Analytical Platforms e.g : dashboards, reports, other applications, etc . )

For fellows from the software development scene, robust CI processes, automated testing, and extensive monitoring are nothing new.   
The last 15 years have brought them countless steps forward.

Solutions such as Jenkins or Gitlab CI for CI/CD, Datadog, ELK stack for aggregated logs and monitoring, Unit / Integration test frameworks automated testing is widely used. We need to build on this knowledge and we need to make up for our lag on the data front.

People smarter than me at Monte Carlo have done a pretty good job of defining the difference between Data Observability and Testing. You can read more about this [here](https://www.montecarlodata.com/blog-what-is-data-observability/#Data-observability-vs.-testing).

From a reliability perspective, my checklist for a robust Data Warehouse:

*   Testing
*   Robust deployment/release process
*   Alerting / Monitoring

### **Testing + and much more testing**

The importance of data tests is unquestionable in data projects. As Analytics Engineers we must ensure that writing the tests is **not** stuck in the backlog purgatory.

Without schema/ integrity tests, eventually, you will run into data failures, which will lead to a loss of confidence in the Analytics Platform.

Fortunately, dbt out-of-the-box supports the ability to test your data from the beginning of development — to the daily scheduled jobs.

The testing starter pack consists of:

*   Uniqueness: To ensure there are no duplicates in a column
*   Not null: To check that there are no NULL values present in a column
*   Relationship: To check the referential integrity between related fields in two model  
    Accepted values: To test that a column only contains a set of values e.g: “credit card”, “debit card”
*   Freshness test on staging tables: To check whether new records are inserted to the table

**Custom test:**

In my opinion, Singular tests is one of the best ways to ensure that things don’t break in the development process. Basically you have to recalculate a KPI that you are sure is not going to change e.g last year’s sales amount. The test will compare a manually calculated hard-coded fix value with the current state and throw an error if something changed.

Example singular test:

```sql
with total\_revenue\_test as (

   select sum(payment\_amt) as payment\_total\_amt

   from {{ref('fct\_payments')}}

   where payment\_dt between '2020-01-01' and '2020-12-31'

),

compare\_result as (  
  select   
      \*  
   from total\_revenue\_test  
    where payment\_total\_amt <> 872342312 -- your hardcoded KPI  
)

select \* from compare\_result

```
In addition, we have to raise attention to potential schema drifting but I will write about it in the following post.

You can read more about the built-in tests [here](https://docs.getdbt.com/docs/building-a-dbt-project/tests), if you can’t find what you’re looking for you can still write a [custom generic test](https://docs.getdbt.com/docs/guides/writing-custom-generic-tests), but before you do that it’s worth looking around in the famous [dbt-expectations](https://github.com/calogica/dbt-expectations) package.

If you are interested in the Unit testing options in dbt, one of my colleagues wrote a detailed how-to guide on this topic. You can find Sonny’s writing [here](https://blog.hiflylabs.hu/en/2022/02/10/environment-dependent-unit-testing-in-dbt/).

Stay tuned for the next episode, where I will write about the Deployment/CI aspect of a robust Data platform.