+++
fragment = "content"
weight = 100

title = "Create your own website"
background = "light"
[sidebar]
  sticky = true
+++

Infrastructure automation for hosting your website
<!--more-->




### Purpose

You might have seen many examples of screen by screen walk-through about creating a website. This blog will help you to create a scalable website. Not just that, This blueprint can help you create a website with just a single execution and can be reproduced many times. We will use [cloudformation](https://aws.amazon.com/cloudformation/) which helps you write [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code). I assume you have some familiarity with AWS services used in this blog and basic concepts about websites and you have an AWS [account](https://console.aws.amazon.com) to try

### You need some HTML code

I used react-helloworld example from [here](https://github.com/ahfarmer/calculator). Follow the instructutions in README.md

```
git clone https://github.com/ahfarmer/calculator.git
npm install
npm run-script build
```
## AWS Resources
You will be creating an S3 bucket to keep your code, a CloudFront distribution.

#### CloudFront Access Identity
We need to restrict the access to your site using an origin access identity so that only cloudfront distribution can access your bucket.

```
  CDNS3AccessId:
    Type: 'AWS::CloudFront::CloudFrontOriginAccessIdentity'
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: 'This Distribution of calculator app'

```

#### S3 Bucket
Amazon Simple Storage Service [Amazon S3](https://aws.amazon.com/s3/) is one of many storage services offered by Amazon. You can store your data in Object format and protect it. It is fast, secure, durable, scalable and reliable and the metrics differ slightly based on the type of storage class.

So, Let's look at how to create an S3 bucket using for our usecase using Cloudformation
```
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      Tags:
        -
          Key: type
          Value: website
```
Now, we need to add some policies so that cloudfront can access your S3 Bucket.
```
  S3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref S3Bucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: CloudFrontAccess
            Effect: 'Allow'
            Action: 
              - s3:GetObject
              - s3:ListBucket
            Principal:
              AWS:
                !Sub 'arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${CDNS3AccessId}'
            Resource: 
              - !Sub '${S3Bucket.Arn}/*'
              - !Sub '${S3Bucket.Arn}'
```

### CloudFront
[AWS Cloudfront] (https://aws.amazon.com/cloudfront/) is a content delivery network like [Akamai](https://www.akamai.com/) or [Cloudfare](https://www.cloudflare.com/). It servers your content securely through it's globally distributed high speed edge networks. A distribution tells how and where to manage and serve the content from.

lets see how to create a Cloudfront a distribution
```
  CloudFrontDistribution:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        DefaultRootObject: !Ref DefaultRootObject
        Enabled: true
        HttpVersion: http2
        PriceClass: PriceClass_100
```
#### Origins
When you create a distribution, you need to specify what is the source of your content. As you know, We have a bucket created for it and we are going to upload our website files there. So, We need to create an Origin for the S3Bucket. Here is how
```
        Origins:
          - DomainName: !Sub '${S3Bucket}.s3.amazonaws.com'
            Id: s3origin
            S3OriginConfig:
              OriginAccessIdentity: !Sub 'origin-access-identity/cloudfront/${CDNS3AccessId}'
```

#### behaviors

You will need to define some request and response behaviors on how cloudfront process your request and response to and from the S3 bucket. The behavior configures information like caching which one one of many features of cloudfront, cookies, Cross origin resource sharing(CORS) etc. Since we have only one bucket, It will be a default behavior. 

```
        DefaultCacheBehavior:
          TargetOriginId: 's3origin'
          DefaultTTL: 300 #if you want to avoid frequently fetching from S3, keep max
          MaxTTL: 280000
          MinTTL: 0
          ForwardedValues:
            QueryString: false
            Cookies:
              Forward: none
          ViewerProtocolPolicy: redirect-to-https
          Compress: true
```
#### Error Response
Defining error response let you define the custom error messages to the user. for ex, user is trying an invalid URL and the origin returns 404 and cloudfront will return the custom error message to the user as defined in the error response.

When you host a static site, It is important to define the error response for 404 user to index.html for routing to work correctly with single page applications using react. You can [refer](### References & Further Readings) for different options for web applications with directory structures

```
        CustomErrorResponses:
        - ErrorCode: 403 # Access denied
          ResponseCode: 200
          ResponsePagePath: '/login.html'
        - ErrorCode: 404 # not found
          ResponseCode: 404
          ResponsePagePath: '/index.html'

```
## Finally!

Now you have the cloudformation ready, You can run it using aws-cli or upload to s3 bucket and run from cloudformation console. 

Once you execute you will be able to view your bucket. Now copy the content of public folder in your localmachine to S3 bucket.

Voila! You will be able to access your website using the cloudfront distribution URL.

### References & Further Readings

- [AWS Samples](https://github.com/aws-samples/)
- [hosting websites with subdirectories](https://stackoverflow.com/questions/31017105/how-do-you-set-a-default-root-object-for-subdirectories-for-a-statically-hosted)
