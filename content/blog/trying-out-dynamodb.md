---
title: Trying out DynamoDB and serverless AWS services in general
description:
date: 2026-02-11
tags:
  - aws
  - dynamodb
draft: true
---

Recently, I've been interested in creating a serverless application with an existing (very) WIP side project I have that's been sitting aroud for a bit. Part of my motivation for learning serverless AWS services is because it seems neat and having less of an operational burden (not that I really have any experience with operating software in production environments), but also it seems like it would be cheaper than paying for a VPS over the course of a year. At the moment (Feb 2026), I'm paying $6/month for a 1GB Digital Ocean droplet running Docker to host other side projects that are using sqlite as databases. This isn't terribly expensive, but if I wanted to run some other rdbms technology, I'd either have to run it in another Docker container (don't particularly want to administer a db myself at the moment, especially one that is a layer removed from running on an actual machine, also don't know if my cheap droplet can handle multiple db containers running at the same time), or pay for a managed db service for an additional monthly cost. I saw that Planetscale has a $5 option for its cheapest managed Postgres offering, but that costs a little more than $0.

# AWS services I think I'll need

- Cloudfront to serve static assets built from a full stack React framework I'd be using (potentially Tanstack Start), coupled with ACM to provide TLS/https when serving the assets
- Route53 for DNS stuff, as well as associating ACM cert with Cloudfront
- S3 to host the actual static assets and act as origin for Cloudfront
- Lambda to run the back end portion of whatever full stack React framework I choose to use. Not entirely sure how it would work here, though. Would the whole app be put into a single Lambda function and be called whenever the front end needs data?
- API Gateway (I assume) to interface between the Internet and my Lambda functions
- DynamoDB for data storage - this is probably the service I'm most interested in of those mentioned here because of the type of data modeling that seems requied for efficient querying, that is, single table design
- Cognito for auth? I honestly have no idea what the auth situation is here. Cognito handles everything? Or still need to store users in DynamoDB?
- others...?

# DynamoDB concepts

- partition key - like a primary key in a relational database, unique value?
- sort key - optional, but can be included to allow for sorting and stuff? The combination of `partition key - sort key` has to be unique, but the sort key doesn't have to be unique across all partitions?
- local secondary index - can only be defined at table creation time, max of 5. Keep same partition key, but use different sort key. Must include existing partition key, but can use a different sort key. The sort key from the base table must be an attribute in the LSI
- global secondary index - can be defined after table creation, can include totally different partition and sort keys from the base table
