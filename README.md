<!-- PROJECT LOGO -->
<br />
<p align="center">
  <a href="https://firefly-iii.org/">
    <img src="https://raw.githubusercontent.com/firefly-iii/firefly-iii/develop/.github/assets/img/logo-small.png" alt="Firefly III" width="120" height="178">
  </a>	&nbsp 	&nbsp 	&nbsp
  <a href="https://n26.com/en-eu">
    <img src="https://www.czerwona-skarbonka.pl/wp-content/uploads/2020/10/n26-logo.png" alt="N26" width="120" height="82">
  </a>
</p>

  <h1 align="center">Firefly III Implementation on N26 landscape</h1>

  <p align="center">
    A free and open source personal finance manager
    <br />
    <a href="https://docs.firefly-iii.org/"><strong>Firefly III official documentation</strong></a>
    <br />
    <br />
  </p>

<!-- MarkdownTOC autolink="true" -->
- [Purpose](#purpose)

<!-- /MarkdownTOC -->

## Purpose

Purpose of this document is to provide both architectural and operational overview of Firefly III setup on N26 environment. This can be a literature for Solution Architects as well as DevOps engineers(AWS CLI and kubectl script files will be provided)

## Prerequisites

Initial configuration that is provided on landscape is that, the corporate domain is managed on Amazon Route53 service and corporate email server runs on Google Workspace(GSuite). Considering that main cloud provider is AWS, all setup will be taken place on AWS environment

## Installation

### Application:
There are several ways of installing FireFly III PHP application layer on AWS:
  - classic way is installing it with all needed PHP libraries on EC2 instance running on Linux2 AMI 
  - more faster way on installation would be using Docker container technology and spin up ECS task either on Fargate(serverless) or EC2 connected to it
  - more advanced way would be starting K8S cluster on EKS backing it up by either Fargate(serverless) or EC2

### Database:
System requires relational database and for setup instead of bearing with infra-hosting and volume/storage management (in case of stateful K8S), I would prefer PAAS by AWS which in our case is Amazon RDS.


## Scalability and Availability
In order to make application reachable on https://firefly3.n26.com domain from anywhere securely with high availability, several AWS can be used such as ELB, ASG, CloudFront and Route53.
 - From ELB services Layer4 (Applciation LB) can be used (no need for Layer7 - NLB) . ALB will be responsible for balancing the load by filtering HTTP requests and route them across machines (target groups) with activated Healthcheck of EC2 instances. Additionally, TLS offloading in order to decrease decryption workload on targets and Sticky Sessions for not losing the session on client side can be enabled on ALB. ALB can be used in 2 hosting methods - on ECS and EC2. In order to balance the load on EKS, AWS Load Balancer Controller add-on should be deployed on your cluster.
 - For horizontal scalability -  
 - For content delivery CloudFront
 - For availability on domain Route53 service should be configured



