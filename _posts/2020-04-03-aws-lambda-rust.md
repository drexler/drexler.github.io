---
layout: post
title: AWS Lambda & Rust 
date: '2020-04-03T09:08:00.000-07:00'
categories: rust
author: dReXler
tags:
- rust
---
#### Background
<p>
The HR product my previous team inherited and migrated from Azure to AWS was built using ASP.Net in VB.Net. As one can imagine, this legacy application although particularly useful is woefully inadequate when 
the modern alternatives such as single-page applications offer a smoother user experience. In order to modernize it, distinct functional parts of the application were to be
re-written with ReactJS and the resulting bundle served out from a Cloudfront-distributed S3 bucket to the application on page loads. At Asure, all new development is <i>Cloud</i>-first. The 
earliest module re-written this way was <i>Direct Deposits</i> whose backend was a series of lambdas utilizing node-mssql to interact with an RDS datastore. 
</p>
<p>
Each tenant had a series of stored encrypted credentials that needed to be decoded for further calls into the other internal applications that
linked the HR application to the Payroll suite. These were decoded on the fly. To replicate this, the original VB.Net code was ported into a utility lambda in .Net Core (2.1)
from which other <i>Direct Deposit</i>-related lambdas could call into.  Ideally, having this a lambda layer would have been nice but with the different runtimes
involved - Node & .Net Core, that was ruled out. 
</p>

#### Problem
With production workloads, each lambda needing to create/update/delete an existing direct deposit needed to await the result of the call to decrypt the necesary credentials.
The associated cold start with the .Net Core-based decryptor lambda became a bottleneck to other lambdas and overall had a noticeable impact on the user experience. Here's 
an image the latency involved: 

![decrypt-lambda-unoptimized](/assets/imgs/decrypt-lambda-unoptimized.png)

#### Solution
There were a few possible solutions:
* Find supporting Node libraries and fold the existing decryption functionality into the various services. 
* Rewrite the decryption lambda in a different runtime. 
  
The first option looked promising, however, the nuances between this specific implementation and Node's were a bit troublesome. The risk to breaking the existing services were also a factor. The second was limited in scope and with the alternatives available: Go & Rust, there was an opportunity to investigate how these languages could be leveraged to meet the performance constraints we sought as well as expand the *tools* available to the team when it comes to performance-related problems. Since Rust is not garbage-collected and offers near native-C style performance, that won out. Admittedly, i am biased when it comes to Rust.

Utilizing [Rusoto](https://github.com/rusoto/rusoto), the [Serverless-Rust](https://github.com/softprops/serverless-rust) plugin, i canaried the Rust-equivalent version of the decryptor service. This was a non-optimized version with
the following traits:
* Non-architecture specific build
* Skipped prewarming of the service investigate cold-start effects
* Used synchronous IO-blocking version of [Rusoto](https://github.com/rusoto/rusoto), version [0.42](https://docs.rs/crate/rusoto_core/0.42.0). 
* OpenSSL instead of Rust-TLS

The result: **7X improvement on cold starts!** 

![rust-decrypt-lambda](/assets/imgs/rust-decrypt-lambda-unoptimized.png)




#### What about .Net Core 3.1?
So a few days back, AWS started providing support for .Net 3.1. Would upgrading to that help with the overall cold start improvement with the decrypt service. I did rummage around with that but although there's a noticeable improvement in the overall cold starts the .Net Core 3.1-based lambda, the unoptimized Rust version still pips it at the post.  Here's a summary of my findings. Note, this is not perfect benchmark but rather a focused use case analysis.

Lambda Runtime | Container Type | Avg Cold Start Time(ms) | Avg Duration(ms) | Avg Memory/Invocation(MB) |
---------------| ---------------|-------------------------| -----------------| --------------------------|
.Net Core 2.1  | AmazonLinux 2  | 4819                    | 115              | 94                        |
Custom (Rust)  | AmazonLinux    | 283                     | 232              | 39                        |
.Net Core 3.1  | AmazonLinux 2  | 3549                    | 94               | 111                       | 
Custom (Rust**)| AmazonLinux    | 203                     | 76               | 34                        |

Rust** : *Partially optimized (Rust TLS for Rusoto SDK, targeted architecture: x86_64-unknown-linux-musl)*





