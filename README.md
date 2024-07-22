# Using AWS Cloudformation to automate the setup of a website hosting on Amazon S3 bucket with CloudFront and Route53
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

1. A CloudFormation template is designed to provision the resources for this solution
2. The stack is created in CloudFormation with the appropriate role adhering to the least privilege principle
 
<br>

1. ![Image description](https://github.com/Sjleal/aws-static-website-cloudfront/blob/main/images/screnshots/diagram/arc-01.png)Developer makes changes to the website and pushes them to the remote repository
2. Using the AWS Connector application on GitHub, the connection to AWS has been established to report changes to the project.
3. A trigger is activated and the source stage of the pipeline is started.
4. The deployment stage starts, which updates the files in the S3 bucket

<br>

1. ![Image description](https://github.com/Sjleal/aws-static-website-cloudfront/blob/main/images/screnshots/diagram/dev-01.png) User makes the request to the website. 
2. Route53, acting as DNS, routes the request to the corresponding Cloudfront distribution
3. ACM is integrated with CloudFront to deploy the ACM certificate on the CloudFront distributions
4. If the requested content can be served from the cache it will be delivered immediately from the edge location closest to the user. If the content is not cached, CloudFront requests the content directly from S3 buckets

   A. The request is for the subdomain bucket, it contains the website and delivered to Cloudfront.
    
   B. The request is for the domain bucket, in this case the request is redirected to the subdomain bucket, and delivered to Cloudfront.

5. Cloudfront gets the updated content from the subdomain bucket.    
6. Cloudfront return the content
7. User gets the response





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
  s3bucketsub:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub ${psub}.${proot}
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
  s3bucketroot:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Ref proot
      WebsiteConfiguration:
        RedirectAllRequestsTo:
          HostName: !Ref s3bucketsub
          Protocol: https
      AccessControl: BucketOwnerFullControl
~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**4. Configuring Route 53**

If you have already registered your domain with AWS, Route 53 automatically creates a hosted zone with the same name. In this case, our domain was registered with an external provider, we need to create the hosted zone in Route 53, copy the NS records generated, and modify the nameservers at the domain register platform. The following code was used to create and configure the resource via cloudformation template:

~~~
Resources:
  r53hostzone:
    Type: 'AWS::Route53::HostedZone'
    Properties:
      HostedZoneConfig: 
        Comment: !Sub Public hosted zone for ${proot}
      Name: !Ref proot
~~~


<br>

**5. Requesting a SSL/TLS certificate**

In order to secure the connection to the website, it is necessary to have an SSL certificate that encrypts the communication between browsers and the website. AWS Certificate Manager service is is responsible for creating, storing and renewing SSL/TLS certificates. The following code was used to request the certificate via cloudformation template:

~~~
Resources:
  sslcert:
    Type: 'AWS::CertificateManager::Certificate'
    Properties:
      DomainName: !Ref proot
      SubjectAlternativeNames:
        - !Sub '*.${proot}'
      DomainValidationOptions:
        - DomainName: !Ref proot
          HostedZoneId: !GetAtt r53hostzone.Id
      ValidationMethod: DNS
~~~

When requesting a certificate, a validation process is initiated. In this case, DNS is selected as the validation method. This process may take some time, after which the certificate will be issued.



<br>

**6. Creating an Origin Access Identity (OAI)**

If you want to restrict access to an Amazon Simple Storage Service origin, you can use an origin access identity (OAI) instead of using public access to the bucket content, which is a potential security threat. The following code was used to create the resource via cloudformation template:

~~~
Resources:
  cfoai:
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
  policys3bucketsub:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Sub '${psub}.${proot}'
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Action: 's3:GetObject'
          Effect: Allow
          Resource: !Sub 'arn:aws:s3:::${psub}.${proot}/*'
          Principal:
            CanonicalUser: !GetAtt cfoai.S3CanonicalUserId
        - Sid: AllowSSLRequestsOnly 
          Effect: Deny
          Principal: '*'
          Action: 's3:*'
          Resource:
          - !Sub 'arn:aws:s3:::${psub}.${proot}'
          - !Sub 'arn:aws:s3:::${psub}.${proot}/*'
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
  cfdistsub:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Comment: CloudFront distribution points to S3 bucket for subdomain
        Origins:
          - DomainName: !Sub '${psub}.${proot}.s3.${AWS::Region}.amazonaws.com'
            Id: !Sub 'S3Origin-${psub}.${proot}'
            S3OriginConfig:
              OriginAccessIdentity: !Sub 'origin-access-identity/cloudfront/${cfoai}'
        Aliases:
          - !Sub '${psub}.${proot}'
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
          TargetOriginId: !Sub 'S3Origin-${psub}.${proot}'
          ViewerProtocolPolicy: 'redirect-to-https'
        DefaultRootObject: 'index.html' 
        Enabled: true
        HttpVersion: http2
        PriceClass: PriceClass_All
        ViewerCertificate:
          AcmCertificateArn: !Ref sslcert
          SslSupportMethod: sni-only
  cfdistroot:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Comment: CloudFront distribution points to S3 bucket for root domain
        Origins: 
          - DomainName: !Sub '${proot}.s3-website-${AWS::Region}.amazonaws.com'
            Id: !Sub 'RedirectS3Origin-${proot}' 
            CustomOriginConfig:
              HTTPPort: 80
              HTTPSPort: 443
              OriginProtocolPolicy: 'http-only' 
        Aliases:
          - !Sub '${proot}'
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
          TargetOriginId: !Sub 'RedirectS3Origin-${proot}'
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
  r53hostzonerecords:
    Type: 'AWS::Route53::RecordSetGroup'
    Properties:
      Comment: Route53 record for CloudFront distributions for root domain and subdomain
      HostedZoneId: !GetAtt r53hostzone.Id
      RecordSets:
        - Name: !Sub '${psub}.${proot}.'
          Type: A
          AliasTarget:
              DNSName: !GetAtt cfdistsub.DomainName
              HostedZoneId: Z2FDTNDATAQYW2
        - Name: !Sub '${proot}.'
          Type: A
          AliasTarget:
              DNSName: !GetAtt cfdistroot.DomainName 
              HostedZoneId: Z2FDTNDATAQYW2
~~~

In AliasTarget you must specify a HostedZoneId, which is an alias resource records sets only with a specific value and depends on where you want to route traffic. In this case, Z2FDTNDATAQYW2, this is always the hosted zone ID when you create an alias record that routes traffic to a CloudFront distribution.

<br>

**9. Creating the stack with Cloudformation**

Once we have finished designing the template for our stack, it is time to build it. As I mentioned before, the complete template is available in a public repository on GitHub called static-website-cloudformation.

You can use a tool named [Application Composer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/app-composer-for-cloudformation.html) in CloudFormation console mode to validate your template and also you can drag, drop, configure, and connect a variety of resources onto a visual canvas.

To [create the stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the AWS CloudFormation console. 
2. Choose Create Stack to start the Create Stack wizard.
3. __Selecting a stack template.__ On the Specify template page, choose Upload a template file to select the CloudFormation template designed.
4. __Specifying stack parameters.__ On the Specify stack details page, type a stack name in the Stack name box. In the Parameters section, specify parameters domain, subdomain, and the unique tag.
5. __Setting AWS CloudFormation stack options.__ On _Permissions - optional_, select a role that allow to this stack create, update or delete the resources involved, __IMPORTANT:__ you can create this role on IAM console but remember using the least privilege principle. In _Stack failure options_ you can set the stack behavior in case of provisioning failure.
6. __Reviewing your stack.__ Here You can review the selected options and press Submmit button to start the creation process.

<br>

---

>**Final steps:**<br>
In this section 


<br>

**10. Creating a pipeline**

Once the resources have been created, we need to upload the files needed to update the website. To do this, we need to create a pipeline that allows us to upload a change to our GitHub repository and automatically update it to the S3 bucket.

To create a [Pipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the CodePipeline console.
2. Choose Pipeline in the left column and click in Create pipeline to start the pipeline wizard.
   1. __Choose pipeline settings.__
        - Pipeline name: _Project-name_
        - Execution mode: Queued
        - Service role: New service role
   2. __Add source stage.__
        - Source provider: GitHub (Version 2)
        - Connection: Connect to GitHub 
          - Connection name: _Project-connection_
          - GitHub Apps: Install a new app
            - Login to [GitHub.com](https://github.com/)
            - Install the AWS Connector for GitHub
            - Repository access: Only selected repositories
        - Repository name: _Project-name_
        - Default branch: main
        - Trigger
          - Trigger type: No filter
   3. __Add build stage.__
        - Skip build stage
   4. __Add deploy stage.__
        - Deploy provider: Amazon S3
        - Bucket: Project-Bucket
        - Extract file before deploy: Check
   5. __Review.__
        - Create pipeline

<br>

**11. Uploading files to website**

Once the steps above are complete, you can make changes to your website code (any HTML, CSS, or Java files) and push the files to your remote repository ([GitHub.com](https://github.com/)). You can then verify in the pipeline that the source and deploy stages were triggered and the changes are uploaded to the repository within seconds. You can then visit your website to verify those changes were applied.