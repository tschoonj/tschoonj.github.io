---
layout: post
title: "Monitoring Ceph buckets with Python"
date: 2021-05-01 13:40:30 +0000
comments: true
author: Tom Schoonjans
categories: [Ceph, S3, AMQP, RabbitMQ, SNS, python, boto3]
---

My [last blogpost]({% post_url 2021-02-20-aws-s3-bucket-monitoring %}) covered how to monitor S3 buckets on Amazon Web Services (AWS) from Python using the [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) library. Today I will be sharing some of the things I learned while working on a very similar topic: monitoring buckets on a _[Ceph storage cluster](https://ceph.io)_. For those who are not familiar with Ceph, it is a massive object store on a distributed computing system, and provides 3-in-1 interfaces for object-, block- and file-level storage. Its reliability is ensured through replication across multiple disks, and is extremely scalable, offering storage in the exabyte range, which explains why [CERN uses it](https://indico.cern.ch/event/765214/contributions/3517140/attachments/1908787/3154067/Echo-TomByrne.pdf) for storing the data obtained from the Large Hadron Collider. Ceph is also [commonly used as storage backend for Openstack](https://docs.ceph.com/en/latest/rbd/rbd-openstack/) deployments.

Currently Ceph offers two mechanisms for users to be notified when an object has been created or deleted in a bucket:

* Pull: through a publish and subscribe mechanism ([PubSub](https://docs.ceph.com/en/latest/radosgw/pubsub-module/)), a user is able to query a topic attached to a bucket for object modification events, using a REST API. Notification events may be S3 structure compliant or not. This mechanism has been deprecated.
* Push: notifications are sent to an external message broker (AMQP 0.9.1, Kafka, or HTTP endpoints) that consumers subscribe through a topic that was attached to one or more buckets. Notification events messages are ([mostly](https://tracker.ceph.com/issues/50115)) compatible with the S3 message schema.

A [third](https://fosdem.org/2021/schedule/event/sds_ceph_rgw_serverless/) mechanism is currently under development that uses the push strategy _without_ the external broker, and will instead rely on an AWS SQS compatible internal message queuing system. I expect that when this mechanism goes into production, the workflow covered in my [previous blogpost]({% post_url 2021-02-20-aws-s3-bucket-monitoring %}), will be applicable to Ceph as well, with minimal changes.

In this blogpost, I will demonstrate how to to use the push mechanism with an AMQP 0.9.1 external broker.

## Before you get started

To try out the following example code and the gist, you will need access to a running Ceph cluster through an access and secret keypair. This could be a Ceph cluster that runs [on your local machine](https://docs.ceph.com/en/latest/dev/quick_guide/#running-a-development-deployment).

You will also need a running AMQP 0.9.1 compatible message broker (I used [RabbitMQ](https://www.rabbitmq.com)), which needs to be reachable by both the Ceph cluster and the machine that you will be monitoring from.

It should be possible to run a Ceph cluster and RabbitMQ broker on the local machine that is also used to run the following Python code from, but I have not tried that.

## Creating the clients

We will be using both the S3 and SNS (for creating topics) services so we need to instantiate two separate clients to interact with them:

```python
BUCKET_NAME = 'test-bucket-' + ''.join(random.choice(string.ascii_lowercase) for i in range(10))
TOPIC_NAME = 's3-bucket-monitor-' + ''.join(random.choice(string.ascii_lowercase) for i in range(10))
AMQP_URL = os.environ['AMQP_URL']
AMQP_EXCHANGE = os.environ['AMQP_EXCHANGE']

client_options = {
    'aws_access_key_id': os.environ['AWS_ACCESS_KEY_ID'],
    'aws_secret_access_key': os.environ['AWS_SECRET_ACCESS_KEY'],
    'region_name': os.environ.get('AWS_REGION', ''),
    'endpoint_url': os.environ.get('CEPH_ENDPOINT_URL'),
}

s3_client = boto3.client('s3', **client_options)
sns_client = boto3.client('sns', **client_options, config=Config(signature_version="s3"))
```

If your bucket was not created in the default region, make sure to set the `AWS_REGION` to the correct region name, as the SNS queue must be created in the same region as the bucket is located in!

## Create the exchange

The following code snippet starts off with initializing a dict containing the attributes necessary to create the topic.
Next, the AMQP exchange mentioned in these attributes is created (if necessary). Note that currently Ceph/RadosGW enforces that topics pointing to the same AMQP endpoint, must use the same exchange, which is rather annoying. Finally, the topic itself is created:

```python
# generate URL query with endpoint_args
endpoint_args = f"push-endpoint={AMQP_URL}&amqp-exchange={AMQP_EXCHANGE}&amqp-ack-level=broker"

# parse it properly
attributes = {
    nvp[0]: nvp[1]
    for nvp in urllib.parse.parse_qsl(
        endpoint_args, keep_blank_values=True
    )
}

# configure the Pika AMQP client
connection_params = pika.connection.URLParameters(AMQP_URL)

# create the exchange
with pika.BlockingConnection(connection_params) as connection:
with connection.channel() as channel:
    channel.exchange_declare(
	exchange=AMQP_EXCHANGE,
	exchange_type="topic",
	durable=True,
    )

# create a topic
resp = sns_client.create_topic(Name=TOPIC_NAME, Attributes=attributes)
topic_arn = resp["TopicArn"]

```

## Configuring the bucket

With the topic now created, the time has come to configure the bucket itself, and instruct it to send events to our AMQP exchange:

```python
topic_conf_list = [
{
    "TopicArn": topic_arn,
    "Events": ["s3:ObjectCreated:*",],
    "Id": "type-here-something-possibly-useful", # Id is mandatory!
},
]

s3_client.put_bucket_notification_configuration(
Bucket=BUCKET_NAME,
NotificationConfiguration={
    "TopicConfigurations": topic_conf_list
},
)
```

In this example we will be looking exclusively at newly created files, but it is also possible to observe files [that were deleted](https://docs.ceph.com/en/latest/radosgw/s3-notification-compatibility/#event-types). Note that the topic configuration must contain an `Id`: while AWS allows users to leave it out and let the endpoint generate one for you, this is not true for Ceph/RadosGW, even if you do not plan to use it further on.

## Create a queue and consume messages

The last step is to launch an infinite while loop that will periodically check for new messages (events) in a newly created queue that will be bound to our exchange. After extracting the filename from the message, we print it, and acknowledge receipt of the message. When run from the command line, you may want to embed the while loop in a `try/except KeyboardInterrupt` block to exit the loop, and switch to using `basic_consume` and `start_consuming, as is done in the gist.`

```python
with pika.BlockingConnection(connection_params) as connection:
    with connection.channel() as channel:
    	result = channel.queue_declare("", exclusive=True)
        queue_name = result.method.queue
        channel.queue_bind(
            exchange=AMQP_EXCHANGE,
            queue=queue_name,
            routing_key=TOPIC_NAME,
        )
        while True:
            method_frame, _, _body = channel.basic_get(queue_name)
            if method_frame:
                body = json.loads(_body)
                channel.basic_ack(method_frame.delivery_tag)
                # we are going to assume 1 record per message
                try:
                    record = body["Records"][0]
                    event_name: str = record["eventName"]
                except Exception as e:
                    logger.info(
                        f"Ignoring {_body=} because of {str(e)}"
                    )
                    continue

                if "ObjectCreated" in event_name:
                    # new file created!
                    s3_info = record["s3"]
                    object_info = s3_info['object']
                    key = urllib.parse.unquote_plus(object_info['key'])
                    logger.info(f'Found new object {key}')
            else:
                time.sleep(1)
```

## What about SSL?

Assuming you have enabled your RabbitMQ broker with SSL certificates (you should!), you will most likely pass an amqps URL to `AMQP_URL`. Unfortunately, this is currently not supported by any Ceph releases. However, I have written a [patch](https://github.com/ceph/ceph/pull/39392/commits/1418bcc1dc3f22257fec840556902b4bf88932b8) that adds support for this, and has been merged into master, so it should be available in the next major release, and has been backported to their Pacific release.

That being said, I would still exercise great caution when using SSL connections between RadosGW and the RabbitMQ broker, as I have observed connection failures when the certificate is changed on the broker. If you're using Letsencrypt certificates like me, this will happen 2-3 months, leaving you with no other option then to restart RadosGW... This should be fixed when somebody addresses [this issue](https://tracker.ceph.com/issues/49033).

## Acknowledgements

I would like to thank [Tom Byrne](https://www.linkedin.com/in/tom-byrne-960016bb/), Storage Systems Admin@STFC, for helping me test and debug the Ceph bucket events mechanism, as well as [Yuval Lifshitz](https://www.linkedin.com/in/yuvallifshitz/), Principal Engineer@Red Hat, for his help and advice, during testing and while getting my patch into the Ceph codebase.

## The gist of it

A complete example, which will create the bucket for you, and upload files to it while monitoring can be found in the following gist. It also includes proper error handling, which I have ignored until now, as well as thorough cleanup of all resources on Ceph/RadosGW.

{% gist 0ef6eed94271dac5685a99f084ea5ed4 %}
