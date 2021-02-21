---
layout: post
title: "Monitoring AWS S3 buckets with Python"
date: 2021-02-20 13:40:30 +0000
comments: true
author: Tom Schoonjans
categories: [AWS, S3, SQS, python, boto3]
---

One of the projects that I have spent a lot of time on over the last couple of months is the [RFI-File-Monitor](https://github.com/rosalindfranklininstitute/rfi-file-monitor), which is a desktop app that we are currently deploying at the [Franklin](https://www.rfi.ac.uk/) to archive, catalogue and process the experimental data that is produced by the scientific instruments. Written in Python, with a GUI built using PyGobject/Gtk+3, its main purpose is to monitor local directories for newly created and modified files, which are then processed using a pipeline consisting of a number of _operations_: currently the user is able to upload the observed files to S3, SFTP, Dropbox, as well as copy them to a local folder (which may be an attached network drive), or compress them into a zipfile or tarball.

Monitoring files in directories is just one of the _engines_ that we have developed to generate the stream of files that is sent to the pipeline: we also support monitoring objects in S3 buckets hosted on both Amazon Web Services (AWS) and Ceph clusters. Since monitoring AWS and Ceph buckets require very different approaches and implementations, we have come up with two separate engines that are available to the user. Both were rather challenging to develop as I couldn't find much relevant sample code online, so I thought it would be useful to share the underlying code that powers both engines.

In this blogpost I will try and explain how one can monitor AWS S3 buckets from Python using the *boto3* library through a complete example. My plan is to follow up soon with a similar post about monitoring Ceph S3 buckets...

## Before you get started

To monitor an S3 bucket on AWS you will need the access and secret keys of an IAM user with an appropriate policy attached to it. I have found that the following minimum policy works well for this purpose:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:PutBucketNotification",
                "s3:ListBucket",
                "s3:GetBucketNotification",
                "sqs:DeleteMessage",
                "sqs:DeleteMessageBatch",
                "sqs:ReceiveMessage",
                "sqs:DeleteQueue",
                "sqs:GetQueueAttributes",
                "sqs:CreateQueue",
                "sqs:SetQueueAttributes"
            ],
            "Resource": [
                "arn:aws:sqs:*:*:s3-bucket-monitor-*",
                "arn:aws:s3:::your-bucket-name/*",
                "arn:aws:s3:::your-bucket-name"
            ]
        }
    ]
}
```

The policy reveals which AWS service we will be using for the monitoring: [Simple Queue Service (SQS)](https://aws.amazon.com/sqs/). By configuring the bucket of interest (_your-bucket-name_) to generate an SQS message for every object event and sending it to an SQS queue we subscribe to, we can be notified of newly created (and deleted) objects!

## Creating the clients

We will be using both the S3 and SQS services so we need to instantiate two separate clients to interact with them:

```python
bucket_name = 'your-bucket-name'

client_options = {
    'aws_access_key_id': os.environ['AWS_ACCESS_KEY_ID'],
    'aws_secret_access_key': os.environ['AWS_SECRET_ACCESS_KEY'],
    'region_name': os.environ.get('AWS_REGION', 'us-east-1'),
}

s3_client = boto3.client('s3', **client_options)
sqs_client = boto3.client('sqs', **client_options)
```

If your bucket was not created in the default `us-east-1` region, make sure to set the `AWS_REGION` to the correct region name, as the SQS queue must be created in the same region as the bucket is located in!

## Create the queue

The following code snippet starts off with generating a unique name for the SQS queue, which is then used to create it. According to the [docs](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sqs.html#SQS.Client.create_queue), one must wait at least 1 second for the queue to become usable so we make the process sleep for that amount of time.

The call that creates the queue returns the _queue URL_, which we need to use to get the corresponding _queue arn_. This parameter is used in the policy that we attach to the queue to ensure that it will be _allowed_ to process events originating from the observed bucket.

```python
queue_name = 's3-bucket-monitor-' + '.'.join(random.choice(string.ascii_lowercase) for i in range(10))

queue_url = sqs_client.create_queue(QueueName=queue_name)['QueueUrl']

time.sleep(1)

queue_arn = sqs_client.get_queue_attributes(QueueUrl=queue_url, AttributeNames=['QueueArn'])['Attributes']['QueueArn']

sqs_policy = {
    "Version": "2012-10-17",
    "Id": "example-ID",
    "Statement": [
        {
            "Sid": "Monitor-SQS-ID",
            "Effect": "Allow",
            "Principal": {
                "AWS":"*"  
            },
            "Action": [
                "SQS:SendMessage"
            ],
            "Resource": queue_arn,
            "Condition": {
                "ArnLike": {
                    "aws:SourceArn": f"arn:aws:s3:*:*:{bucket_name}"
                },
            }
        }
    ]
}

sqs_client.set_queue_attributes(
    QueueUrl=queue_url,
    Attributes={
        'Policy': json.dumps(sqs_policy),
    }
)
```

## Configuring the bucket

Now that the queue has been created, and it has been given sufficient privileges to accept events coming from our bucket, the time has come to configure the bucket itself, and instruct it to send events to our queue:

```python
bucket_notification_config = {
    'QueueConfigurations': [
        {
            'QueueArn': queue_arn,
            'Events': [
                's3:ObjectCreated:*',
            ]
        }
    ],
}
s3_client.put_bucket_notification_configuration(
    Bucket=bucket_name,
    NotificationConfiguration=bucket_notification_config
)
```

In this example we will be looking exclusively at newly created files, but it is also possible to observe files [that were deleted or have been restored](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html).

## Subscribe to the queue

The last step is to launch an infinite while loop that will periodically check for new messages (events) in the queue and fetch them. After extracting the filename from the message, we print it, and delete the message from the queue. When run from the command line, you may want to embed the while loop in a `try/except KeyboardInterrupt` block to exit the loop. 

```python
while True:
    resp = sqs_client.receive_message(
        QueueUrl=queue_url,
        AttributeNames=['All'],
        MaxNumberOfMessages=10,
        WaitTimeSeconds=10,
        )

    if 'Messages' not in resp:
        print('No messages found')
        continue

    for message in resp['Messages']:
        body = json.loads(message['Body'])
        # we are going to assume 1 record per message
        try:
            record = body['Records'][0]
            event_name = record['eventName']
        except Exception as e:
            print(f'Ignoring {message=} because of {str(e)}')
            continue

        if event_name.startswith('ObjectCreated'):
            # new file created!
            s3_info = record['s3']
            object_info = s3_info['object']
            key = urllib.parse.unquote_plus(object_info['key'])
            print(f'Found new object {key}')

    # delete messages from the queue
    entries = [{'Id': msg['MessageId'], 'ReceiptHandle': msg['ReceiptHandle']} for msg in resp['Messages']]

    resp = sqs_client.delete_message_batch(
        QueueUrl=queue_url, Entries=entries
    )
```

## The gist of it

A complete example, which will create the bucket for you, and upload files to it while monitoring can be found in the following gist. It also includes proper error handling, which I have ignored until now, as well as thorough cleanup of all resources from your AWS account.

{% gist 43102477b1ccac6b1a24e0077a7ff285 %}
