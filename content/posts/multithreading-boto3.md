---
title: "Downloading files from S3 with multithreading and Boto3"
date: 2021-04-09T18:33:17-03:00
draft: false
---

Yesterday I found myself googling how to do something that I'd think it was pretty standard: How to download multiple files from [AWS](https://aws.amazon.com/) [S3](https://aws.amazon.com/s3/) in parallel using Python?

After not finding anything reliable in [Stack Overflow](https://stackoverflow.com/), I went to the Boto3 [documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) and started coding. Something I thought it would take me like 15 mins, ended up taking me a couple of hours. I'm writing this post with the hope I can help somebody who needs to perform this basic task and possibly me in the future.

This blogpost is organized as follows, if you're just looking for the working snippet go to [3](#solution-with-tqdm-working-with-multiple-threads):

1. [Boto3 in a nutshell: clients, sessions and resources](#boto3-in-a-nutshell-clients-sessions-and-resources)
2. [Is Boto3 threadsafe?](#is-boto3-threadsafe)
3. [Solution (with tqdm working with multiple threads)](#solution-with-tqdm-working-with-multiple-threads)
---

## Boto3 in a nutshell: clients, sessions, and resources
Boto3 is the official Python SDK for accessing and managing all AWS resources. Generally it's pretty straightforward to use but sometimes it has weird behaviours, and its documentation can be confusing.

Its 3 most used features are: sessions, clients, and resources.

### Session
"A session manages state about a particular configuration. By default, a session is created for you when needed. However, it's possible and recommended that in some scenarios you maintain your own session.", according to the [docs](https://boto3.amazonaws.com/v1/documentation/api/1.14.31/guide/session.html). Basically it's an object that stores all the relevant configuration you'll need to interact with AWS. You'll mainly be setting the IAM credentials and region with it.
```python
import boto3

# creating a session with the default settings
session = boto3.Session()

# creating another session for a different region
session_other_region = boto3.Session(region_name='us-west-2)
```

Once you have a session, there are two different APIs to interact with AWS: client (low level) and resource (high level)

### Client
A client provides a low level API to interact with the AWS resources. You can perform *all* available AWS actions with a client.
```python
# Get all S3 buckets using the client API
s3 = session.client("s3")
s3.list_buckets()
```

I just used sessions and clients but for completeness let's review the resources as well.

### Resource
A resource is a high level representation available for *some* AWS resources such as S3. It can lead to simpler code. 
```python
# Get all S3 buckets using the resource API
s3 = session.resource("s3")
s3.buckets.all()
```

You can also can directly instantiate a resource and a client, without explicitly defining a session. If you do something like this:
```python
s3 = boto3.resource("s3")
```
you are just using the default session which loads all the default settings.

---

## Is Boto3 threadsafe?
{{< highlight text>}}TL;DR clients are threadsafe but sessions and resources are not.{{< /highlight>}}

Although the [documentation](https://boto3.amazonaws.com/v1/documentation/api/1.14.31/guide/session.html#multithreading-or-multiprocessing-with-sessions) specifically addresses this question, it's not clear. Luckily I'm not the only one who found it confusing, this [issue](https://github.com/boto/botocore/issues/1246) from 2017 still remains active and open.

The docs say:

>"It is recommended to create a resource instance for each thread / process in a multithreaded or multiprocess application rather than sharing a single instance among the threads / processes. For example:"

```python
import boto3
import boto3.session
import threading

class MyTask(threading.Thread):
    def run(self):
        session = boto3.session.Session()
        s3 = session.resource('s3')
        # ... do some work with S3 ...
```
>"In the example above, each thread would have its own Boto3 session and its own instance of the S3 resource. This is a good idea because resources contain shared data when loaded and calling actions, accessing properties, or manually loading or reloading the resource can modify this data.

>"Similar to Resource objects, Session objects are not thread safe and should not be shared across threads and processes. You should create a new Session object for each thread or process:"
```python
import boto3
import boto3.session
import threading

class MyTask(threading.Thread):
    def run(self):
        # Here we create a new session per thread
        session = boto3.session.Session()

        # Next, we create a resource client using our thread's session object
        s3 = session.resource('s3')

        # Put your thread-safe code here
```
Based on that explanation, I understood that I had to create a new session (and client) in each thread. Doing that ends up being quite slow, since creating a session has some overhead.

Actually you can create only one session and one client and pass that client to each thread, see the [note](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/resources.html#multithreading-and-multiprocessing). Since clients are thread safe there's nothing wrong with that. However, if you need to interact with the session object, you need to create one session in each thread, that wasn't the case.

Here's a quick test to compare the two solutions, sharing the client between threads vs creating a new session and client in each thread:

```python
def list_all_buckets_new_client():
    session = boto3.Session()
    client = session.client('s3')
    client.list_buckets()

def list_all_buckets_one_client(client):
    client.list_buckets()

def shared_client():
    session = boto3.Session()
    client = session.client('s3')
    with ThreadPoolExecutor(max_workers=8) as executor:
        futures = [executor.submit(list_all_buckets_one_client, client) for i in range(100)]

def multiple_clients():
     with ThreadPoolExecutor(max_workers=8) as executor:
         futures = [executor.submit(list_all_buckets_new_client) for i in range(100)]

%timeit shared_client()
3.05 s ± 101 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

%timeit multiple_clients()
12.2 s ± 562 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

---

## Solution (with tqdm working with multiple threads)
Context: I needed to download multiple zip files from one single S3 bucket.

Requirement: I wanted to do it in parallel since each download was independent from the others.

Bonus: Have a progress bar using [tqdm](https://github.com/tqdm/tqdm) that updated on each download.

{{< highlight text>}}TL;DR instantiate one client and pass it to each thread. {{< /highlight >}} 

Although all of this seems pretty standard, it wasn't easy to find a working solution neither for using multithreading with Boto3 or using multithreading with tqdm. I hope this is useful for somebody out there having this (also) pretty standard problem.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from functools import partial
import os

import boto3
import tqdm

AWS_BUCKET = "my-bucket"
OUTPUT_DIR = "downloads"

def download_one_file(bucket: str, output: str, client: boto3.client, s3_file: str):
    """
    Download a single file from S3
    Args:
        bucket (str): S3 bucket where images are hosted
        output (str): Dir to store the images
        client (boto3.client): S3 client
        s3_file (str): S3 object name
    """
    client.download_file(
        Bucket=bucket, Key=s3_file, Filename=os.path.join(output, s3_file)
    )


files_to_download = ["file_1", "file_2", ..., "file_n"]
# Creating only one session and one client
session = boto3.Session()
client = session.client("s3")
# The client is shared between threads
func = partial(download_one_file, AWS_BUCKET, OUTPUT_DIR, client)

# List for storing possible failed downloads to retry later
failed_downloads = []

with tqdm.tqdm(desc="Downloading images from S3", total=len(files_to_download)) as pbar:
    with ThreadPoolExecutor(max_workers=32) as executor:
        # Using a dict for preserving the downloaded file for each future, to store it as a failure if we need that
        futures = {
            executor.submit(func, file_to_download): file_to_download for file_to_download in files_to_download
        }
        for future in as_completed(futures):
            if future.exception():
                failed_downloads.append(futures[future])
            pbar.update(1)
if len(failed_downloads) > 0:
    print("Some downloads have failed. Saving ids to csv")
    with open(
        os.path.join(OUTPUT_DIR, "failed_downloads.csv"), "w", newline=""
    ) as csvfile:
        wr = csv.writer(csvfile, quoting=csv.QUOTE_ALL)
        wr.writerow(failed_downloads)
```
---
Thanks for reading!

I hope you now understood which features of Boto3 are threadsafe and which are not, and most importantly, that you learned how to download multiple files from S3 in parallel using Python.

If you have any doubts or comments, reach out on [Twitter](https://twitter.com/emasquil) or by [email](mailto:emasquildev@gmail.com).