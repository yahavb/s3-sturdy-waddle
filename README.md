# How to grant access to S3 bucket 

Assumptions:
Objects are already in S3. We created a bucket called `t2demo` and populated with few files in the bucket's root. 

Request: 
Enable authenticated users to download S3 objects similar to SFTP methods. 

We will review four options and discuss tradeoffs

## Methods:
### 1 - Authenticated AWS users with appropriate roles can download objects from the S3 console, AWS CLI, AWS SDK. 

AWS CLI is probably the closest to sftp e.g., `aws s3 ls s3://t2demo/` and `aws s3 cp s3://t2demo/bit.ly_3zdRwgd.png` 

##### Pros - Bucket uses default hardened configuration 
##### Cons - Need to grant proper role and attach it to every user

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::t2demo/*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::t2demo"
        },
    ]
}
```

More in 
https://aws.amazon.com/blogs/security/writing-iam-policies-how-to-grant-access-to-an-amazon-s3-bucket/

### 2 - adding user specific folders permissions
Extending the policy from 1/ and add `s3:ListBucket` per folder. E.g., lets assume the following bucket structure:

```
->t2demo
->  /user1/
->  /user2/
...
->  /user3/
```

user1...user3 denotes bucket sub folders that has different objects. Granting a user access to a subfolder requires to narrow the `s3:ListBucket` permission. e.g.,

Here we grant `list` permission to the entire bucket (`arn:aws:s3:::t2demo`) 

```json
{
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::t2demo"
        },
```

Instead we add more specific `Condition` e.g.,

```json
{
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::t2demo",
            "Condition":{"StringLike":{"s3:prefix":["/user1/*"]}
}
``` 
#### Pros - Most granular permission settings.
#### Cons - Requires extra permission management.  

### 3 - Use ABAC to scale the user specifics with ABAC - requires SSO 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::t2demo/*",
            "Condition": {
                "StringLike": {
                    "s3:ExistingObjectTag/team": "prod"
                }
            }
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::t2demo",
            "Condition": {
                "StringLike": {
                    "s3:ExistingObjectTag/team": "prod"
                }
            }
        }
    ]
}
```

#### Pros - Simplifies option 3, i.e., no need for user specific IAM config
#### Cons - Complexity - Requires SSO integration 

More in 
https://docs.aws.amazon.com/IAM/latest/UserGuide/access_tags.html

### 4 - pre-signed url and provide encoded https request per content.

```bash
aws s3 presign s3://t2demo/bit.ly_3zdRwgd.png
```
Copy the output to a browser and get the object


#### Pros - Most secure from bucket policy perspectives, no need to manage user access I.e., zero extra IAM roles and policies, easy to manage, only one more extra step (see below)
#### Cons - urls are for objects only, no list bucket objects. presigned urls are time bounded. Not working for permanent access.   


More in
https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/presign.html
https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html


