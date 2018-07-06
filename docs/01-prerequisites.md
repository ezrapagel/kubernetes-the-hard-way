# Prerequisites

## Google Cloud Platform

This tutorial leverages the [Google Cloud Platform](https://cloud.google.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://cloud.google.com/free/) for $300 in free credits.

[Estimated cost](https://cloud.google.com/products/calculator/#id=78df6ced-9c50-48f8-a670-bc5003f2ddaa) to run this tutorial: $0.22 per hour ($5.39 per day).

> The compute resources required for this tutorial exceed the Google Cloud Platform free tier.

## Google Cloud Platform SDK

### Install the Google Cloud SDK

Follow the Google Cloud SDK [documentation](https://cloud.google.com/sdk/) to install and configure the `gcloud` command line utility.

Verify the Google Cloud SDK version is 200.0.0 or higher:

```
gcloud version
```

### Set a Default Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

If you are using the `gcloud` command-line tool for the first time `init` is the easiest way to do this:

```
gcloud init
```

Otherwise set a default compute region:

```
gcloud config set compute/region us-west1
```

Set a default compute zone:

```
gcloud config set compute/zone us-west1-c
```

> Use the `gcloud compute zones list` command to view additional regions and zones.

## Amazon Web Services

This tutorial leverages [Amazon Web Services](https://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://aws.amazon.com/free/) for AWS Free Tier.

**Estimated** cost to run this tutorial: $0.35 per hour ($8.50 per day), based on my experience (YMMV).

> The compute resources required for this tutorial exceed the AWS Free Tier.

## AWS Command Line Interface

### Install the AWS Command Line Interface

Follow the AWS Command Line Interface [documentation](https://docs.aws.amazon.com/cli/latest/userguide/installing.html) to install and configure the `aws` command line utility.

Verify the AWS Command Line Interface version is 1.15.39 or higher:

```
aws --version
```

### Set a Default Compute Region and Zone

This tutorial assumes a default compute region has been configured.

If you are using the `aws` command-line tool for the first time `configure` is the easiest way to do this:

```
aws configure
```

Otherwise set a default compute region:

```
aws configure set region us-east-1
```

> Use the `aws ec2 describe-regions` and `aws ec2 describe-availability-zones --region region-name` command to view additional regions and zones.


## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)