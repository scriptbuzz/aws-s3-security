# aws-s3-security
Controls I use often to secure S3 data by enforcing rules such as IP sources, VPC origins, encryption, MFA, etc. NOTE: Resource names, resource IDs, and IP addressed in these examples are for illustration purposes and will be different in your account.

- White list IP ranges: Deny API actions on an S3 bucket and its objects unless from specified IP addresses in the Condition block of the policy.
```
{
  "Version": "2012-10-17",
  "Id": "Example",
  "Statement": [
    {
      "Sid": "Ex1a",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
	       "arn:aws:s3:::mys3bucket",
         "arn:aws:s3:::mys3bucket/*"
      ],
      "Condition": {
	 "NotIpAddress": {"aws:SourceIp": "203.0.113.0/24"}
      }
    }
  ]
}
```

- White list and blacklist IP ranges: Alow API actions on  S3 bucket objects when API action originates from specified IP addresses in  IpAddress entry of the policy, but deny if originated from NotIpAddress IP ranges.
```
{
  "Id":"PolicyId2",
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"Example",
      "Effect":"Allow",
      "Principal":"*",
      "Action":"s3:*",
      "Resource":"arn:aws:s3:::mys3bucket/*",
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24"},
        "NotIpAddress": { "aws:SourceIp": "54.0.143.128/30"}
      }
    }
  ]
}
```
- Enforce MFA: Deny any operation on an S3 bucket unless authintcated with MFA.
```
{
    "Version": "2012-10-17",
    "Id": "Example",
    "Statement": [
      {
        "Sid": "Ex1a",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": "arn:aws:s3:::mys3bucket/applogs/*",
        "Condition": { "Null": { "aws:MultiFactorAuthAge": true }}
      }
    ]
 }
```
- Bucket owner must have full control of objects uploaded to S3 bucket from an external AWS account. 

Example policy

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Example",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ExternalAccountID:user/developer01"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::mys3bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-grant-full-control": "id=SourceAaccount-CanonicalUserID"
        }
      }
    }
  ]
}
```
Example of CLI command from external account used to give bucket owner in source account full control of created S3 objects:
```
aws s3api put-object-acl --bucket mys3bucket --key keyname --acl bucket-owner-full-control
```
- Limit S3 bucket creation to a specific region.
```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Sid":"Example",
         "Effect":"Allow",
         "Action": "s3:CreateBucket",
         "Resource": "arn:aws:s3:::*",
         "Condition": {
             "StringLike": {
                 "s3:LocationConstraint": "us-east-1"
             }
         }
       }
    ]
}
```
- Limit access to an S3 bucket and its content to a VPC endpoint. Note: This will disable console access to the S3 bucket
```
{
   "Version": "2012-10-17",
   "Id": "Example",
   "Statement": [
     {
       "Sid": "Ex1a",
       "Principal": "*",
       "Action": "s3:*",
       "Effect": "Deny",
       "Resource": ["arn:aws:s3:::mys3bucket",
                    "arn:aws:s3:::mys3bucket/*"],
       "Condition": {
         "StringNotEquals": {
           "aws:SourceVpce": "vpce-123456"
         }
       }
     }
   ]
}
```
- Limit access to an S3 bucket and its content to a specified VPC
```
{
   "Version": "2012-10-17",
   "Id": "Example",
   "Statement": [
     {
       "Sid": "Ex1a",
       "Principal": "*",
       "Action": "s3:*",
       "Effect": "Deny",
       "Resource": ["arn:aws:s3:::mys3bucket",
                    "arn:aws:s3:::mys3bucket/*"],
       "Condition": {
         "StringNotEquals": {
           "aws:SourceVpc": "vpc-123456"
         }
       }
     }
   ]
}
```
- Enforce Encryption In Transit: Deny access to an S3 bucket if https is not used for API actions
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [ "12345678912"]
      },
      "Action": "s3:Get*",
      "Resource": "arn:aws:s3:::mys3bucket/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Resource": "arn:aws:s3:::mys3bucket/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```
- Enforce SSE-S3 Encryption At Rest: Deny any Put actions that do not have a header to encrypt the object using SSE-S3 encryption.
```
{
     "Version": "2012-10-17",
     "Id": "Example",
     "Statement": [
           {
                "Sid": "Ex1a",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3:::mys3bucket/*",
                "Condition": {
                        "StringNotEquals": { "s3:x-amz-server-side-encryption": "AES256" }
                }
           },
           {
                "Sid": "Ex1b",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3:::mys3bucket/*",
                "Condition": {
                        "Null": { "s3:x-amz-server-side-encryption": true }
               }
           }
     ]
 }
```
- Enforce SSE-KMS Encryption At Rest: Deny any Put actions that do not have a header to encrypt the object using SSE-KMS encryption.
```
{
       "Version": "2012-10-17",
       "Id": "Example",
       "Statement": [
           {
                "Sid": "Ex1a",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3:::mys3bucket/*",
                "Condition": {
                    "StringNotEquals": {
                          "s3:x-amz-server-side-encryption": "aws:kms"
                             }
                   }
           },
           {
                "Sid": "Ex1b",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:PutObject",
                "Resource": "arn:aws:s3::mys3bucket/*",
                "Condition": {
                    "Null": {
                          "s3:x-amz-server-side-encryption": true
                            }
                    }
           }
    ]
}
```
