# Using AWS Cloudformation to automate the setup of a website hosting on Amazon S3 bucket with CloudFront + Route53
## Overview

This project offers a solution to a specific scenario, deploying a static website to an Amazon S3 bucket and distributing it with Amazon CloudFront. With Aws CloudFormation, it was possible to automate the creation and provision of AWS infrastructure deployments, using template files to create or delete resources in a single unit as a stack. To meet the goal of this project, the steps were executed: 

#### Initial setup
+ Step 1. **Creating a static webstite.**
+ Step 2. **Registering a custom domain.**
#### CloudFormation template 
+ Step 3. **Creating S3 buckets.**
+ Step 4. **Configuring Route 53.**
+ Step 5. **Requesting a SSL/TLS certificate.**
+ Step 6. **Creating an Origin Access Identity (OAI).**
+ Step 7. **Creating a policy to allow OAI to access bucket content.**
+ Step 8. **Setting up CloudFront distributions.**
+ Step 9. **Routing DNS traffic using Route 53.**
+ Step 10. **Creating the stack.**
#### Final steps
+ Step 10. **Creating a pipeline.**
+ Step 11. **Uploading files to website.**

As you can see, most of the steps will be automated in an AWS CloudFormation template, which will be explained throughout this document. While the final steps will be configured manually on AWS and GitHub, the goal is the same: to automate the process of updating the code and deploying the resources used to host the website.

## Architecture

The following image provides a view of the proposed architecture for deploying website resources.



In this case, the static website consist of a mix if HTML documents, images, CCS stylesheets, and JavaScript files, which will be store on an S3 bucket. Taking advantage of the S3 service, an endpoint will be created, and it can be used for Cloudfront to distribute the website.

## Services

- [GitHub](https://github.com/) is a developer platform that allows developers to create, store, manage and share their code.
- [Amazon S3](https://docs.aws.amazon.com/s3/) is a cloud-based object storage service that helps you store, protect, and retrieve any amount of data.
- [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/iam/) helps you securely manage access to your AWS resources by controlling who is authenticated and authorized to use them.
- [AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/) helps you create, store, and renew public and private SSL/TLS X.509 certificates and keys that protect your AWS websites and applications.
- [Amazon Route 53](https://docs.aws.amazon.com/route53/) is a highly available and scalable DNS web service.
- [Amazon CloudFront](https://docs.aws.amazon.com/cloudfront/) speeds up distribution of your web content by delivering it through a worldwide network of data centers, which lowers latency and improves performance.
- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/) helps you set up AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You can use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually. You can manage and provision stacks across multiple AWS accounts and AWS Regions.
- [AWS Codepipeline](https://docs.aws.amazon.com/codepipeline/) helps you quickly model and configure the different stages of a software release and automate the steps required to release software changes continuously.

## Step by Step

>**Initial setup:**<br>
In these initial steps, we will focus on preparing the website source code, uploading it to our repository, and reserving the domain name. This setup process will be run almost entirely on external platforms, but AWS has products that can provide these facilities.



**1. Creating a static webstite**

- Create or use a pre-built basic html website template
- Create a folder named _proyect_name_ and copy any HTML documents, images, CCS stylesheets, and JavaScript files
- Setup your authentication method on GitHub [Authentication to GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github#authenticating-with-the-command-line)
- Create a new repository on GitHub: [Creating a new repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository)
- Setup your remote repository: [Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
- Modify the necessary files to update website and Commit changes to the GitHub repository. 

**2. Registering a custom domain**

You can register a domain name on [Amazon Route 53](https://docs.aws.amazon.com/route53/) or [Namecheap](https://www.namecheap.com//) or [GoDaddy](https://www.godaddy.com/domains/) or [Hostinger](https://www.hostinger.com/) or any domains registrars of your choice.  

<br>

---

>**CloudFormation Template:**<br>
I am an automation enthusiast and I believe Infrastructure as Code (Iac) is the best way to maintain infrastructure integrity, reduce errors, speed up deployments, troubleshoot issues, and track changes over time.<br> 
Taking advantage of Cloudformation's ability to automate resource deployment, a template will be designed to handle the creation and configuration of the resources involved in the solution. In the following steps, part of the code used in each section will be shown, the complete template is available in a public repository called static-website-cloudformation.<br>
Some variables will be requested at stack creation and others will be captured during template execution. The YAML format was chosen for this template.


<br>

**3. Creating S3 buckets**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  s3bucket-sub:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub ${p_sub}.${p_root}
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
        - BucketKeyEnabled: true
          ServerSideEncryptionByDefault:
            SSEAlgorithm: "AES256"
  s3bucket-root:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Ref p_root
      WebsiteConfiguration:
        RedirectAllRequestsTo:
          HostName: !Ref s3bucket-sub
          Protocol: https
      AccessControl: BucketOwnerFullControl
~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**4. Configuring Route 53**

If you have already registered your domain with AWS, Route 53 automatically creates a hosted zone with the same name. In this case, our domain was registered with an external provider, we need to create the hosted zone in Route 53, copy the NS records generated, and modify the nameservers at the domain register platform. The following code was used to create and configure the resource via cloudformation template:

~~~
Resources:
  r53host-zone:
    Type: 'AWS::Route53::HostedZone'
    Properties:
      HostedZoneConfig: 
        Comment: !Sub Public hosted zone for ${p_root}
      Name: !Ref p_root
~~~


<br>

**5. Requesting a SSL/TLS certificate**

In order to secure the connection to the website, it is necessary to have an SSL certificate that encrypts the communication between browsers and the website. AWS Certificate Manager service is is responsible for creating, storing and renewing SSL/TLS certificates. The following code was used to request the certificate via cloudformation template:

~~~
Resources:
  sslcert:
    Type: 'AWS::CertificateManager::Certificate'
    Properties:
      DomainName: !Ref p_root
      SubjectAlternativeNames:
        - !Sub '*.${p_root}'
      DomainValidationOptions:
        - DomainName: !Ref p_root
          HostedZoneId: !GetAtt r53host-zone.Id
      ValidationMethod: DNS
~~~

When requesting a certificate, a validation process is initiated. In this case, DNS is selected as the validation method. This process may take some time, after which the certificate will be issued.



<br>

**6. Creating an Origin Access Identity (OAI)**

If you want to restrict access to an Amazon Simple Storage Service origin, you can use an origin access identity (OAI) instead of using public access to the bucket content, which is a potential security threat. The following code was used to create the resource via cloudformation template:

~~~
Resources:
  cf-oai:
    Type: 'AWS::CloudFront::CloudFrontOriginAccessIdentity'
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: 'OAI for S3 origins'
~~~

After that we can use this resource into the bucket policy to prevent others from access the S3 content by using the website endpoint.

<br>

**7. Create a policy to allow OAI to access bucket content**

Now it's time to add a policy to the website bucket and grant access only to the newly created resource (OAI) and prevent any attempt to establish non-SSL/TLS communication. The following code was used to add the described bucket policy via cloudformation template:

~~~
Resources:
  policy-s3bucket-sub:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Sub '${p_sub}.${p_root}'
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Action: 's3:GetObject'
          Effect: Allow
          Resource: !Sub 'arn:aws:s3:::${p_sub}.${p_root}/*'
          Principal:
            CanonicalUser: !GetAtt cf-oai.S3CanonicalUserId
        - Sid: AllowSSLRequestsOnly 
          Effect: Deny
          Principal: '*'
          Action: 's3:*'
          Resource:
          - !Sub 'arn:aws:s3:::${p_sub}.${p_root}'
          - !Sub 'arn:aws:s3:::${p_sub}.${p_root}/*'
          Condition:
            Bool:
              'aws:SecureTransport': false
~~~


<br>

**8. Setting up CloudFront distributions**

Amazon CloudFront is a web service used for content caching, which speeds up content delivery across a global network. In this section, two Cloudfront distributions will be created, one for the website bucket and one for the bucket that redirects traffic to https.

The distribution for the website bucket is of the S3OriginConfig type and the redirect bucket distribution is of the CustomOriginConfig type. Additionally, a series of parameters are configured such as CustomErrorResponses, DefaultCacheBehavior, AcmCertificateArn and others that define the behavior of the distributions.

The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  cf-dist-sub:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Comment: CloudFront distribution points to S3 bucket for subdomain
        Origins:
          - DomainName: !Sub '${p_sub}.${p_root}.s3.${AWS::Region}.amazonaws.com'
            Id: !Sub 'S3Origin-${p_sub}.${p_root}'
            S3OriginConfig:
              OriginAccessIdentity: !Sub 'origin-access-identity/cloudfront/${cf-oai}'
        Aliases:
          - !Sub '${p_sub}.${p_root}'
        CustomErrorResponses:
          - ErrorCode: 403
            ResponseCode: 404
            ResponsePagePath: '/error.html'
            ErrorCachingMinTTL: 60
        DefaultCacheBehavior:
          AllowedMethods:
            - GET
            - HEAD
            - OPTIONS
          CachedMethods:
            - GET
            - HEAD
            - OPTIONS
          Compress: true
          DefaultTTL: 3600
          ForwardedValues:
            QueryString: false
            Cookies:
              Forward: none
          MaxTTL: 86400
          MinTTL: 60
          TargetOriginId: !Sub 'S3Origin-${p_sub}.${p_root}'
          ViewerProtocolPolicy: 'redirect-to-https'
        DefaultRootObject: 'index.html' 
        Enabled: true
        HttpVersion: http2
        PriceClass: PriceClass_All
        ViewerCertificate:
          AcmCertificateArn: !Ref sslcert
          SslSupportMethod: sni-only
  cf-dist-root:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Comment: CloudFront distribution points to S3 bucket for root domain
        Origins: 
          - DomainName: !Sub '${p_root}.s3-website-${AWS::Region}.amazonaws.com'
            Id: !Sub 'RedirectS3Origin-${p_root}' 
            CustomOriginConfig:
              HTTPPort: 80
              HTTPSPort: 443
              OriginProtocolPolicy: 'http-only' 
        Aliases:
          - !Sub '${p_root}'
        CustomErrorResponses:
          - ErrorCode: 403
            ResponseCode: 404
            ResponsePagePath: '/error.html'
            ErrorCachingMinTTL: 60
        DefaultCacheBehavior:
          AllowedMethods:
            - GET
            - HEAD
            - OPTIONS
          CachedMethods:
            - GET
            - HEAD
            - OPTIONS
          Compress: true
          DefaultTTL: 3600
          ForwardedValues:
            QueryString: false
            Cookies:
              Forward: none
          MaxTTL: 86400
          MinTTL: 60
          TargetOriginId: !Sub 'RedirectS3Origin-${p_root}'
          ViewerProtocolPolicy: 'redirect-to-https'
        DefaultRootObject: 'index.html' 
        Enabled: true
        HttpVersion: http2
        PriceClass: PriceClass_All
        ViewerCertificate:
          AcmCertificateArn: !Ref sslcert
          SslSupportMethod: sni-only
~~~


<br>

**9. Routing DNS traffic using Route 53**

Now you need to create record set to route DNS traffic for domain and subdomain to CloudFront distributions. Alias records in Route 53 provide a seamless mapping between your domain name and the CloudFront distribution. When a user accesses your domain, Route 53 automatically routes the request to the corresponding CloudFront distribution, which then serves the content from its edge locations. The following code was used to create both records via cloudformation template:

~~~
Resources:
  r53host-zone-records:
    Type: 'AWS::Route53::RecordSetGroup'
    Properties:
      Comment: Route53 record for CloudFront distributions for root domain and subdomain
      HostedZoneId: !GetAtt r53host-zone.Id
      RecordSets:
        - Name: !Sub '${p_sub}.${p_root}.'
          Type: A
          AliasTarget:
              DNSName: !GetAtt cf-dist-sub.DomainName
              HostedZoneId: Z2FDTNDATAQYW2
        - Name: !Sub '${p_root}.'
          Type: A
          AliasTarget:
              DNSName: !GetAtt cf-dist-root.DomainName 
              HostedZoneId: Z2FDTNDATAQYW2
~~~

In AliasTarget you must specify a HostedZoneId, which is an alias resource records sets only with a specific value and depends on where you want to route traffic. In this case, Z2FDTNDATAQYW2, this is always the hosted zone ID when you create an alias record that routes traffic to a CloudFront distribution.

<br>

**9. Creating the stack with Cloudformation**

Once we have finished designing the template for our stack, it is time to build it. As I mentioned before, the complete template is available in a public repository on GitHub called static-website-cloudformation.

You can use a tool named [Application Composer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/app-composer-for-cloudformation.html) in CloudFormation console mode to validate your template and also you can drag, drop, configure, and connect a variety of resources onto a visual canvas.

To create the stack you need to follow these steps:

1. Logging in to the AWS Management console and open the AWS CloudFormation console. 
2. Choose Create Stack to start the Create Stack wizard. [How to](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html)
3. Selecting a stack template. On the Specify template page, choose Upload a template file to select the CloudFormation template designed.
4. Specifying stack parameters. On the Specify stack details page, type a stack name in the Stack name box. In the Parameters section, specify parameters domain, subdomain, and the unique tag.
5. Setting AWS CloudFormation stack options
6. Reviewing your stack

<br>

---

>**Final steps:**<br>
In this section 


<br>

**3. Creating a pipeline**


1. Register a domain.

2. Create a static html website.
   - Create a Github repository
   - Create a ssh key and upload to github account to commit changes

3. Create a pipeline in Codepipeline.
   - Choose pipeline settings:
     - Pipeline name: Project-name
     - Execution mode: Queued
     - Service role: New service role
   - Add source stage
     - Source provider: GitHub (Version 2)
     - Connection: Connect to GitHub 
       - Connection name: Project-connection
       - GitHub Apps: Install a new app
         - Login to GitHub
         - Install the AWS Connector for GitHub
         - Repository access: Only selected repositories
     - Repository name: Project-name
     - Default branch: main
     - Trigger
       - Trigger type: No filter
   - Add build stage
     - Skip build stage
   - Add deploy stage
     - Deploy provider: Amazon S3
     - Bucket: Project-Bucket
     - Extract file before deploy: Check
   - Review
     - Create pipeline

4. 