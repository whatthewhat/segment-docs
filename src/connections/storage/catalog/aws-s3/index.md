---
title: AWS S3 with IAM Role Support Destination
redirect_from:
  - '/connections/destinations/catalog/aws-s3/'
hide-personas-partial: true
---

> info "This document is about a destination which is in beta"
> This means that the AWS S3 with IAM Role Support destination is in active development, and some functionality may change before it becomes generally available.


## Differences between the Amazon S3 destination and the AWS S3 destination

The AWS S3 destination provides a more secure method of connecting to your S3 buckets. It uses AWS's own IAM Roles to define access to the specified buckets. For more information about IAM Roles, see Amazon's [IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html){:target="_blank"} documentation.

Functionally, the two destinations (Amazon S3 and AWS S3 with IAM Role Support) copy data in a similar manner.

## Getting Started

The AWS S3 destination puts the raw logs of the data Segment receives into your S3 bucket, encrypted, no matter what region the bucket is in.

> info ""
> Segment copies data into your bucket every hour around the :40 minute mark. You may see multiple files over a period of time depending on the amount of data Segment copies.

Keep in mind that AWS S3 works differently than most other destinations. Using a destinations selector like the [integrations object](/docs/connections/spec/common/#integrations) does not affect events with AWS S3.

The diagram below illustrates how the S3 destination works.

The Segment Tracking API processes data from your sources, and collects the Events in batches. When these batches reach a 100 MB, or once per hour, a Segment initiates a process which uploads them to a secure Segment S3 bucket, from which they are securely copied to your own S3 bucket.

![](images/s3processdiagram.png)



## Create a new destination

Complete the following steps to configure the AWS S3 Destination with IAM Role Support.

### Create an IAM role in AWS

To complete this section, you need access to your AWS dashboard.

1. Create a new S3 bucket in your preferred region. For more information, see Amazon's documentation, [Create your first S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html){:target="_blank"}. 
2. Create a new IAM role for Segment to assume. For more information, see Amazon's documentation, [Creating a role to delegate permissions to an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html){:target="_blank"}.
3. Attach the following trust relationship document. Be sure to add your Workspace ID to the `sts:ExternalId` field. 
    ```json
    {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "",
           "Effect": "Allow",
           "Principal": {
             "AWS": "arn:aws:iam::595280932656:role/segment-s3-integration-production-access"
           },
           "Action": "sts:AssumeRole",
           "Condition": {
             "StringEquals": {
               "sts:ExternalId": "<YOUR_WORKSPACE_ID>"
             }
           }
         }
       ]
     }
    ```
4. Create and attach the following IAM policy to the role created in step 3 above. Replace `<YOUR_BUCKET_NAME>` with the name of the bucket you created in step 1 above.
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Sid": "PutObjectsInBucket",
        "Effect": "Allow",
        "Action": [
            "s3:PutObject",
            "s3:PutObjectAcl"
        ],
        "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/segment-logs/*"
        }
    ]
    }
    ```
    If you're using KMS encryption on your S3 bucket, add the following policy to the IAM role:
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowKMS",
            "Effect": "Allow",
            "Action": [
                "kms:GenerateDataKey",
                "kms:Decrypt"
            ],
            "Resource": "<YOUR_KEY_ARN>"
        }
    ]
    }
    ```

If you have server-side encryption enabled, see the [required configuration](#encryption).


### Add the AWS S3 with IAM Role Support Destination

To finish configuration, enable the AWS S3 Destination with IAM Role Support destination in your workspace.

1. Add the **AWS S3** destination from the Raw Data section of the Destinations catalog. This document is about the **AWS S3** destination. For information about the **Amazon S3** destination, which does not include IAM Role support, see the documentation [here](/docs/connections/storage/catalog/amazon-s3/).
2. Select the data source you'll connect to the destination.
3. Provide a unique name for the destination.
4. Complete the destination settings:
   1. Enter the name of the region in which the bucket you created above resides.
   2. Enter the name of the bucket you created above. Be sure to enter the bucket's **name** and not URI.
   3. Enter the ARN of the IAM role you created above. The ARN should follow the format `arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME.`
5. Enable the destination.
6. Verify Segment data is stored in the S3 bucket by navigating to the `<your_S3_bucket>/segment-logs` in the AWS console. The bucket will take roughly 1 hour to begin receiving data.

> info ""
> Did you know you can create destinations with the Config API? For more information, see [Create Destination](https://reference.segmentapis.com/#51d965d3-4a67-4542-ae2c-eb1fdddc3df6){:target="_blank"}.


## Migrate an existing destination

> warning "Avoid overwriting data"
> Sending data to the same S3 location from both the existing Amazon S3 destination, and the AWS S3 with IAM Role Support destination will overwrite data in that location. To avoid this, follow the steps below.

To migrate an existing Amazon S3 destination to the AWS S3 with IAM Role Support Destination:

1. Configure the IAM role and IAM policy permissions as described in steps 2 - 4 [above](#create-an-iam-role-in-aws).
2. Add the AWS S3 with IAM Role Support Destination and add the AWS Region and IAM role ARN. For the bucket name, enter `<YOUR_BUCKET_NAME>/segment-logs/test`. Enable the destination, and verify data is received at `<YOUR_BUCKET_NAME>/segment-logs/test/segment-logs`. If the folder receives data, continue to the next step. If you don't see log entries, check the trust relationship document and IAM policy attached to the role.
3. Update the bucket name in the new destination to `<YOUR_BUCKET_NAME>`.
4. After 1 hour, disable the original Amazon S3 destination.
5. Verify that the `<YOUR_BUCKET_NAME>/segment-logs` receives data.
6. Remove the test folder created in step 2 from the bucket.


### Migration steps for scenarios with multiple sources per environment

In cases where you have multiple sources per environment, for example staging sources pointing to a staging bucket, and production sources going to a production bucket, you need two IAM roles, one for staging, and one for production. 

For example:

- stage_source_1 → stage_bucket
- stage_source_2 → stage_bucket
- stage_source_N → stage_bucket
- prod_source_1 → prod_bucket
- prod_source_2 → prod_bucket
- prod_source_N → prod_bucket

For each source in the scenario, complete the steps described in [Migrate an existing destination](#migrate-an-existing-destination), and ensure that you have separate IAM Roles and Permissions set for staging and production use.


## Data format

Segment stores logs as gzipped, newline-separated JSON containing the full call information. For a list of supported properties, see the [Segment Spec](/docs/connections/spec/) documentation.

Segment groups logs by day, and names them using the following format:

    s3://{bucket}/segment-logs/{source-id}/{received-day}/filename.gz

The received-day refers to the UTC date unix timestamp, that the API receives the file, which makes it easy to find all calls received within a certain timeframe.

## Encryption

Configure encryption at the bucket-level from within the AWS console. For more information, see Amazon's documentation [Protecting data using encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html){:target="_blank"}.

## Custom Path Prefix

To use a custom key prefix for the files in your bucket, append the path to the bucket name in the Segment S3 destination configuration UI. For example, a bucket string `mytestbucket/path/prefix` would result in data copying to `/path/prefix/segment-logs/{source-id}/{received-day}/`.

### How can I download the data from my bucket?

Amazon provides several methods to download data from an S3 bucket. For more information, see [Downloading an object](https://docs.aws.amazon.com/AmazonS3/latest/userguide/download-objects.html){:target="_blank"}.


## Personas

> warning ""
> As mentioned above, the AWS S3 destination works differently than other destinations in Segment. As a result, Segment sends **all** data from a Personas source to S3 during the sync process, not only the connected audiences and traits.

You can send computed traits and audiences generated using [Segment Personas](/docs/personas) to this destination as a **user property**. 

For user-property destinations, Segment sends an [identify](/docs/connections/spec/identify/) call to the destination for each user added and removed. The property name is the snake_cased version of the audience name, with a true/false value to indicate membership. For example, when a user first completes an order in the last 30 days, Personas sends an Identify call with the property `order_completed_last_30days: true`. When the user no longer satisfies this condition (for example, it's been more than 30 days since their last order), Personas sets that value to `false`.

When you first create an audience, Personas sends an Identify call for every user in that audience. Later audience syncs send updates for users whose membership has changed since the last sync.

{% include content/destination-footer.md %}