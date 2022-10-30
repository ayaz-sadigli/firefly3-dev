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
- [Prerequisites](#Prerequisites)
- [Architectural Overview](#architectural-overview)
- [Hosting](#Hosting)
- [Scalability and Availability](#scalability-and-availability)
- [Networking](#Networking)
- [Monitoring and logging](#monitoring-and-logging)
- [Corporate user authentication and authorization](#Corporate-user-authentication-and-authorization)
- [Data Import](#Data-Import)
- [Conclusion](#Conclusion)

 

<!-- /MarkdownTOC -->

## Purpose

Purpose of this document is to provide both architectural and operational overview of Firefly III setup on N26 environment. This can be a literature for Solution Architects as well as DevOps engineers(AWS CLI and kubectl script files will be provided)

## Prerequisites

Initial configuration that is provided on landscape is that, the corporate domain is managed on Amazon Route53 service and corporate email server runs on Google Workspace(GSuite). Considering that main cloud provider is AWS, all setup will be taken place on AWS environment

## Architectural overview
Below Shared Diagram represents "classic" way of installation of Firefly III application on N26 landscape
<br />
<img src="https://github.com/ayaz-sadigli/firefly3-dev/blob/main/N26-classic-setup.png" alt="Firefly III - N26 - Classic" width="1800" height="700">

## Hosting

### Application:
There are several ways of installing FireFly III PHP application layer on AWS:
  - classic way is installing it with all needed PHP libraries on EC2 instance running on Linux2 AMI 
  - more faster way on installation would be using Docker container technology and spin up ECS task either on Fargate(serverless) or EC2 connected to it
  - more advanced way would be starting K8S cluster on EKS backing it up by either Fargate(serverless) or EC2

### Database:
System requires relational database and for setup instead of bearing with infra-hosting and volume/storage management (in case of stateful K8S), PAAS by AWS is preferable which in our case is Amazon RDS. For resilency DB is replicated to standby db and covered by RDS proxy in order to send the traffic to right database. For security purposes, database credentials are stored in AWS Secrets Manager.


## Scalability and Availability
In order to make application reachable on https://firefly3.n26.com domain from anywhere securely with high availability, several AWS can be used such as ELB, ASG, CloudFront/AWS Global Accelerator and Route53.
 - From ELB services Layer4 (Application LB) can be used (no need for Layer7 - NLB). ALB will be responsible for balancing the load by filtering HTTP requests and route them across machines (target groups) with activated Healthcheck of EC2 instances. Additionally, TLS offloading in order to decrease decryption workload on targets and Sticky Sessions for not losing the session on client side can be enabled on ALB. ALB can be used in 2 hosting methods - on ECS and EC2. In order to balance the load on EKS, AWS Load Balancer Controller add-on should be deployed on your cluster.
 - For horizontal scalability -  ASG (Autoscaling Groups) can be implemented on top of EC2 instances and it will ensure min&max&desired number of instances based on metrics you selected on CloudWatch alarms(with step scaling policy). This feature can be used on all hosting methods with a little bit different configurations
 - (Optional) For global availability in lesser latency AWS Global Accelerator can be used on top of ALB. It will also add extra security layer against DDOS attacks
 - For availability on N26.com domain Route53 service should be configured and domainname of our ALB/Global Accelerator should be added as a new alias record.

## Networking



## Monitoring and logging 



## Corporate user authentication and authorization
### Authentication
In order to enable user authentication securely and with less user interaction SSO principles can be used. Custom SAML app on Google Workspace(GSuite) can be created and used as IdP, whereas on application side SP endpoints should be exposed. User will be authenticated based his/her records on GSuite directory

### Authorization
Application will receive and process SAML request from GSuite and based on attributes user role will be assigned to user with required privileges inside the application. To achieve this, application role names can be created as groups in GSuite directory and by this way only authorized users inside the company will access the application


## Data Import



## Conclusion



