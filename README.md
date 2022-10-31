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
- [Networking](#networking)
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
<img src="https://github.com/ayaz-sadigli/firefly3-dev/blob/main/N26-classic-setup.png" alt="Firefly III - N26 - Classic" width="2000" height="500">

## Hosting

### Application:
There are several ways of installing FireFly III PHP application layer on AWS:
  - classic way is installing it with all needed PHP libraries on EC2 instance running on Linux2 AMI 
  - more faster way on installation would be using Docker container technology and spin up ECS task either on Fargate(serverless) or EC2 connected to it
  - more advanced way would be starting K8S cluster on EKS backing it up by either Fargate(serverless) or EC2

#### *In our example we will use classic approach by hosting php application on Apache server installed on EC2 Linux machine.

<details><summary>Configuration details for application hosting</summary>
<p>
Let's assume that you already finished networking setup.
  
#### SSH key generation
  
```
    aws ec2 create-key-pair \
    --key-name demo-key-pair \
    --output text \
    --query “KeyMaterial” \
    --region eu-central-1 > ./demo-key-pair.pem
```  
#### EC2 launch
  
```
     aws ec2 run-instances \
    --image-id <ami-id> \
    --count 1 \
    --instance-type t2.nano \
    --key-name demo-key-pair \
    --security-group-ids <security group id> \
    --subnet-id <subnet id> \
    --user-data file://launch-script.txt
```
#### Launch Script (User-data) (Reference - https://gist.github.com/Engr-AllanG/34e77a08e1482284763fff429cdd92fa)
  
```
#!/bin/bash  
yum update
yum upgrade
yum install nginx curl -y
yum install software-properties-common
#add-yum-repository ppa:ondrej/php
#yum update
yum install -y php7.4 php7.4-{cli,zip,gd,fpm,json,common,mysql,zip,mbstring,curl,xml,bcmath,imap,ldap,intl}
#Ngnix server will replace Apache, that's why we need to disable it
systemctl stop apache2
systemctl disable apache2
#Backup ngnix file
cd /etc/nginx/sites-enabled/
mv default{,.bak}
#Create firefly.conf in sites-enabled folder and then paste in the config below
cat <<EOT >> /etc/nginx/sites-enabled/firefly.conf
server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        #server_name  subdomain.domain.com;
        root         /var/www/html/firefly-iii/public;
        index index.html index.htm index.php;

        location / {
                try_files $uri /index.php$is_args$args;
                autoindex on;
                sendfile off;
       }

        location ~ \.php$ {
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_read_timeout 240;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_split_path_info ^(.+.php)(/.+)$;
        }

    }
EOT
#Restart
systemctl restart nginx php7.4-fpm
#Install firefly - change the version number at the end of the command below to whatever the latest is
cd /var/www/html/
composer create-project grumpydictator/firefly-iii --no-dev --prefer-dist firefly-iii 5.4.6
cd /var/www/html/firefly-iii
php artisan migrate:refresh --seed
php artisan firefly-iii:upgrade-database
php artisan passport:install
  
#Restart
systemctl restart nginx php7.4-fpm

```
</p>
</details>

### Database:
System requires relational database and for setup instead of bearing with infra-hosting and volume/storage management (in case of stateful K8S), PAAS by AWS is preferable which in our case is Amazon RDS. For resilency DB is replicated to standby db and covered by RDS proxy in order to send the traffic to right database. For security purposes, database credentials are stored in AWS Secrets Manager.


## Scalability and Availability
In order to make application reachable on https://firefly3.n26.com domain from anywhere securely with high availability, several AWS can be used such as ELB, ASG, CloudFront/AWS Global Accelerator and Route53.

  - For horizontal scalability -  ASG (Autoscaling Groups) can be implemented on top of EC2 instances and it will ensure min&max&desired number of instances based on metrics you selected on CloudWatch alarms(with step scaling policy).
  
    <details><summary>Configuration details for Auto Scaling Groups</summary>
    <p>
    Let's assume that you already finished networking and application hosting setup. 

    #### Create AMI of current instance

    ```
    aws ec2 create-image --instance-id <Instance Id> --name ASGCLI
    #You should save ImageId in a separate file for later usage.  
    ```  

    #### Create Launch template

    ```
    aws ec2 create-launch-template \
        --launch-template-name TemplateWebServer \
        --version-description AutoScalingVersion1 \
        --launch-template-data '{"NetworkInterfaces":[{"DeviceIndex":0,"AssociatePublicIpAddress":false,"Groups":["sg-08d4e7f2321254357"],"DeleteOnTermination":false}],"ImageId":"ami-04b81c294769991d8","InstanceType":"t2.micro","TagSpecifications":[{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"WebServerASG"}]}]}' \
        --region eu-central-1
    ```    

    #### Create Auto Scaling Group

    ```
    aws autoscaling create-auto-scaling-group \
        --auto-scaling-group-name Firefly3-ASG \
        --launch-template "LaunchTemplateName=TemplateWebServer" \
        --min-size 2 --max-size 5 --desired-capacity 2 
        --availability-zones "eu-central-1a" "eu-central-1b"
    ```      

    </p>
    </details>  
  
  - From ELB services Layer4 (Application LB) can be used (no need for Layer7 - NLB). ALB will be responsible for balancing the load by filtering HTTP requests and route them across machines (autoscaling group/target groups) with activated Healthcheck of EC2 instances. Additionally, TLS offloading in order to decrease decryption workload on targets and Sticky Sessions for not losing the session on client side can be enabled on ALB.
  
    <details><summary>Configuration details for ALB</summary>
    <p>
    Let's assume that you already finished networking, application hosting and autoscaling setup. 

    #### Create ALB 
      
    ```
      aws elbv2 create-load-balancer --name Firefly3-ALB  \
      --subnets subnet-public1 subnet-public2 --security-groups sg-alb
    ```

    #### Create Target Group with default check settings for a TCP taget group
      
    ```
    aws elbv2 create-target-group --name firefly3-tg --protocol HTTP --port 80 \
    --vpc-id vpc-main --ip-address-type [ipv4]
    #You'll receive output with ARN of Target Group which you should save for later usage.  
    ```      

    #### Register target on ALB
      
    ```            
    aws elbv2 register-targets --target-group-arn targetgroup-arn  \
    --targets Id=INSTANCEID
    ```   
      
    #### Create listener on ALB (with TLS - assuming that we already have certificate)
      
    ```            
    aws elbv2 create-listener --load-balancer-arn loadbalancer-arn \
    --protocol HTTPS --port 443  \
    --certificates CertificateArn=CERTIFICATE_ARN \
    --default-actions Type=forward,TargetGroupArn=TARGET_GROUP_ARN
    ```         
     
    #### Attach Load balancer with Autoscaling group
      
    ```             
    aws attach-load-balancers \
    --auto-scaling-group-name Firefly3-ASG
    --load-balancer-names Firefly3-ALB
    ```               
      
    </p>
    </details> 
  
 - (Optional) For global availability in lesser latency AWS Global Accelerator can be used on top of ALB. It will also add extra security layer against DDOS attacks
 
  - For availability on N26.com domain Route53 service should be configured and domainname of our ALB/Global Accelerator should be added as a new alias record.
    
    <details><summary>Configuration details for Route53 update</summary>
    <p>
    Let's assume that you already finished networking, application hosting, autoscaling and load balancing setup. 
     
    #### Create alias resource record set
            
    ```  
    {
      "Comment": "Creating Alias resource record sets in Route 53",
      "Changes": [
        {
          "Action": "CREATE",
          "ResourceRecordSet": {
            "Name": "firefly3.n26.com",
            "Type": "A",
            "AliasTarget": {
              "HostedZoneId": "Z1H1FL5HABSF5",
              "DNSName": "ALB-xxxxxxxx.us-west-2.elb.amazonaws.com",
              "EvaluateTargetHealth": false
            }
          }
        }
      ]
    }      
    ```      
    #### Create alias resource record set
            
    ```  
    aws route53 change-resource-record-sets --hosted-zone-id ZXXXXXXXXXX --change-batch file://recordset.json      
    ```      
    
    </p>
    </details> 
  
## Networking
Due to the application will be hosted in single region multi AZ environment, certain network configurations shoud be met on VPC creation.


## Monitoring and logging 



## Corporate user authentication and authorization
### Authentication
In order to enable user authentication securely and with less user interaction SSO principles can be used. Custom SAML app on Google Workspace(GSuite) can be created and used as IdP, [Google workspace documentation](https://support.google.com/a/answer/6087519?hl=en) can be referred for Custom SAML app setup. On application side Access URL and Entity ID should be exposed. User will be authenticated based his/her records on GSuite directory with SAML 2.0 specification. The sequence diagram below offers more specificity to SP-initiated login process:
<br />
<img src="https://github.com/ayaz-sadigli/firefly3-dev/blob/main/Auth-N26-classic-setup.PNG" alt="Firefly III - Google Workspace" width="700" height="450">


#### *Due to lack of SAML SSO libraries on Firefly III, currently, this configuration needs development on application side

#### Sample SAML request&response should like this:
<details><summary>Request</summary>
<p>
  
#### SAML2.0 Request

```
<saml:AuthnRequest xmlns:saml="urn:oasis:names:tc:SAML:2.0:protocol"
                   AssertionConsumerServiceURL="https://firefly3.n26.com/auth/sso"
                   ForceAuthn="false"
                   ID="xxxx"
                   IsPassive="false"
                   IssueInstant="xxxx"
                   ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
                   Version="2.0"
                   >
    <saml2:Issuer xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">Google IDP url</saml2:Issuer>
    <saml2p:NameIDPolicy xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol"
                         AllowCreate="true"
                         Format="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified"
                         />
    <saml2p:RequestedAuthnContext xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol"
                                  Comparison="exact"
                                  >
        <saml:AuthnContextClassRef xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml:AuthnContextClassRef>
    </saml2p:RequestedAuthnContext>
</saml:AuthnRequest>
```
 
</p>
</details>
<br/>
<details><summary>Response</summary>
<p>
    
#### SAML2.0 Response
  
```
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                ID=""
                Version="2.0"
                IssueInstant=""
                Destination="https://firefly3.n26.com/auth/sso"
                InResponseTo=""
                >
    <Issuer xmlns="urn:oasis:names:tc:SAML:2.0:assertion">Google IDP url</Issuer>
    <samlp:Status>
        <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
    </samlp:Status>
    <Assertion xmlns="urn:oasis:names:tc:SAML:2.0:assertion"
               ID=""
               IssueInstant=""
               Version="2.0"
               >
        <Issuer>Google IDP url</Issuer>
        <Subject>
            <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified">Ayaz</NameID>
            <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                <SubjectConfirmationData InResponseTo=""
                                         NotOnOrAfter=""
                                         Recipient="https://firefly3.n26.com/auth/sso"
                                         />
            </SubjectConfirmation>
        </Subject>
        <Conditions NotBefore=""
                    NotOnOrAfter=""
                    >
            <AudienceRestriction>
                <Audience>@IDP url</Audience>
            </AudienceRestriction>
        </Conditions>
        <AttributeStatement>
            <Attribute Name="mail">
                <AttributeValue>ayaz.sadigli@n26.com</AttributeValue>
            </Attribute>
            <Attribute Name="groups">
                <AttributeValue>CHANGE_REPETITIONS</AttributeValue>
                <AttributeValue>CHANGE_PIGGY_BANKS</AttributeValue>
            </Attribute>
        </AttributeStatement>
        <AuthnStatement AuthnInstant=""
                        SessionIndex=""
                        >
            <AuthnContext>
                <AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</AuthnContextClassRef>
            </AuthnContext>
        </AuthnStatement>
    </Assertion>
</samlp:Response>
```
</p>
</details>

### Authorization
Application will receive and process SAML request from GSuite and based on attributes user role will be assigned to user with required privileges inside the application. To achieve this, application role names can be created as groups in GSuite directory and by this way only authorized users inside the company will access the application. Currently, there are [8 user roles](#roles--group-names) in application side which should be created as groups with same name on Google Workspace directory as well, this [documentation](https://support.google.com/a/users/answer/9303222?hl=en) can be referred on implementation.


#### Roles = Group Names:
- READ_ONLY
- CHANGE_TRANSACTIONS
- CHANGE_RULES
- CHANGE_PIGGY_BANKS
- CHANGE_REPETITIONS
- VIEW_REPORTS
- FULL
- OWNER

#### Access Settings on GW Groups:
<br />
<img src="https://github.com/ayaz-sadigli/firefly3-dev/blob/main/GoogleWorkspace-Groups-N26.PNG" alt="Google Workspace Groups" width="700" height="450">



## Data Import



## Conclusion



