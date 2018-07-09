# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones) or [availability zone](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-regions-availability-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network GCP

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```bash
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```bash
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Virtual Private Cloud Network AWS

In this section a dedicated [Virtual Private Cloud](https://aws.amazon.com/vpc/) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```bash
vpcId=`aws ec2 create-vpc \
  --cidr-block 10.240.0.0/24 \
  --amazon-provided-ipv6-cidr-block \
  --query 'Vpc.VpcId' \
  --output text`
aws ec2 create-tags --resources $vpcId --tag Key=Name,Value=kubernetes-the-hard-way
```

A [subnet](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html#vpc-subnet-basics) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```bash
subnetId=`aws ec2 create-subnet \
  --vpc-id $vpcId \
  --cidr-block 10.240.0.0/24 \
  --query 'Subnet.SubnetId' \
  --output text`
aws ec2 create-tags --resources $subnetId --tag Key=Name,Value=kubernetes-the-hard-way_subnet
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules GCP

Create a firewall rule that allows internal communication across all protocols:

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```bash
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> output

```bash
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Firewall Rules AWS

Create a security group that allows internal communication across all protocols:

```bash
kubernetes_internal_id=`aws ec2 create-security-group \
  --group-name kubernetes-the-hard-way-allow-internal \
  --description kubernetes-the-hard-way-allow-internal \
  --vpc-id $vpcId \
  --query 'GroupId' \
  --output text`
```

```bash
#Remote access (management)
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol icmp --port -1 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol tcp  --port 22 --cidr 0.0.0.0/0
#Internal cluster communication
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol icmp --port -1 --cidr 10.240.0.0/24
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol tcp  --port 0-65535 --cidr 10.240.0.0/24
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol udp  --port 0-65535 --cidr 10.240.0.0/24
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol icmp --port -1 --cidr 10.200.0.0/16
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol tcp  --port 0-65535 --cidr 10.200.0.0/16
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol udp  --port 0-65535 --cidr 10.200.0.0/16
```

Create a security group rule that allows external SSH, ICMP, and HTTPS:

```bash
kubernetes_external_id=`aws ec2 create-security-group \
  --group-name kubernetes-the-hard-way-allow-external \
  --description kubernetes-the-hard-way-allow-external \
  --vpc-id $vpcId \
  --query 'GroupId' \
  --output text`
```

```bash
aws ec2 authorize-security-group-ingress --group-id $kubernetes_external_id --protocol icmp --port -1 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $kubernetes_external_id --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $kubernetes_external_id --protocol tcp --port 6443 --cidr 0.0.0.0/0
```

> An [external load balancer](https://aws.amazon.com/elasticloadbalancing/) will be used to expose the Kubernetes API Servers to remote clients. Allow connections between the ELB and the controller instances (it will actually allow connectivity to worker intances as well, not a biggie for now).

```bash
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol icmp --port -1 --source-group $kubernetes_external_id
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol tcp --port 22 --source-group $kubernetes_external_id
aws ec2 authorize-security-group-ingress --group-id $kubernetes_internal_id --protocol tcp --port 6443 --source-group $kubernetes_external_id
```

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```bash
aws ec2 describe-security-groups --filter Name=vpc-id,Values=$vpcId --output table
```

> output

```console
-------------------------------------------------------------
|                  DescribeSecurityGroups                   |
+-----------------------------------------------------------+
||                     SecurityGroups                      ||
|+--------------+------------------------------------------+|
||  Description |  kubernetes-the-hard-way-allow-external  ||
||  GroupId     |  sg-066dc4c0e694c049f                    ||
||  GroupName   |  kubernetes-the-hard-way-allow-external  ||
||  OwnerId     |  020997832382                            ||
||  VpcId       |  vpc-092329d28410c9db0                   ||
|+--------------+------------------------------------------+|
|||                     IpPermissions                     |||
||+----------------------------------+--------------------+||
|||  FromPort                        |  6443              |||
|||  IpProtocol                      |  tcp               |||
|||  ToPort                          |  6443              |||
....
```

### Kubernetes Public IP Address GCP

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```bash
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```bash
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```console
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

### Kubernetes Public IP Address AWS

Create an elastic load balancer fronting the Kubernetes API Servers:

```bash
KUBERNETES_ELB_DNS=`aws elb create-load-balancer \
  --load-balancer-name kubernetes \
  --listeners Protocol=tcp,LoadBalancerPort=6443,InstanceProtocol=tcp,InstancePort=6443 \
  --subnets $subnetId --security-groups $kubernetes_external_id \
  --query DNSName \
  --output text`
```

```bash
KUBERNETES_PUBLIC_ADDRESS=$KUBERNETES_ELB_DNS
```

Verify the elastic load balancer was created in your default compute region:

```bash
aws elb describe-load-balancers --query 'LoadBalancerDescriptions[].DNSName'
```

> output

```console
[
    "kubernetes-XXXXXXXXXX.us-east-1.elb.amazonaws.com"
]
```

## Compute Instances GCP

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```bash
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```bash
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```bash
gcloud compute instances list
```

> output

```console
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```bash
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```bash
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```bash
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```bash
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-gcp x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```bash
$USER@controller-0:~$ exit
```
> output

```console
logout
Connection to XX.XXX.XXX.XXX closed
```

## Compute Instances AWS

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Upload SSH Key

Create one if you don't have one already. See this [example](https://docs.aws.amazon.com/cli/latest/reference/ec2/import-key-pair.html#examples)

```bash
aws ec2 import-key-pair \
  --key-name "Kubernetes" \
  --public-key-material file://~/.ssh/id_rsa.pub
AWS_SSH_KEY=$(aws ec2 describe-key-pairs --query KeyPairs[0].KeyName --output text)  
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```bash
for i in 0 1 2; do
  declare controller_id${i}=`aws ec2 run-instances \
    --key-name $AWS_SSH_KEY \
    --security-group-ids $kubernetes_internal_id \
    --associate-public-ip-address \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' \
    --image-id ami-39397a46 \
    --private-ip-address 10.240.0.1${i} \
    --subnet-id $subnetId \
    --instance-type t2.medium \
    --query 'Instances[0].InstanceId' \
    --output text`
  ct_id="controller_id$i"
  aws ec2 create-tags \
    --resources ${!ct_id} \
    --tag Key=Name,Value=controller-${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

(*) Seems you can't pass metadata (`pod-cidr`) to AWS instances. One alternative would be to pass it as initial script with `--user-data file://myscript`.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```bash
for i in 0 1 2; do
  declare worker_id${i}=`aws ec2 run-instances \
    --key-name $AWS_SSH_KEY \
    --security-group-ids $kubernetes_internal_id \
    --associate-public-ip-address \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeType":"gp2", "VolumeSize": 200}}]' \
    --image-id ami-39397a46 \
    --private-ip-address 10.240.0.2${i} \
    --subnet-id $subnetId \
    --instance-type t2.medium \
    --query 'Instances[0].InstanceId' \
    --output text`
  wk_id="worker_id$i"
  aws ec2 create-tags \
    --resources ${!wk_id} \
    --tag Key=Name,Value=worker-${i}
done
```

Each EC2 instance performs source/destination checks by default. This means that the instance must be the source or destination of any traffic it sends or receives. However, a NAT instance must be able to send and receive traffic when the source or destination is not itself. Therefore, you must [disable source/destination]((https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck)) checks on the NAT instance.

```bash
for i in 0 1 2; do
  wk_id="worker_id$i"
  ct_id="controller_id$i"
  aws ec2 modify-instance-attribute --instance-id ${!ct_id} --no-source-dest-check
  aws ec2 modify-instance-attribute --instance-id ${!wk_id} --no-source-dest-check
done
```

### Verification

List the compute instances in your default compute zone:

```bash
aws ec2 describe-instances \
  --filter Name=vpc-id,Values=$vpcId \
  --query 'Reservations[].Instances[].[PublicIpAddress,PrivateIpAddress,InstanceId,Tags[?Key==`Name`].Value[]]' \
  --output text
```

> output

```console
XXX.XX.XXX.XX    10.240.0.11    i-09734066f00792f9e
controller-1
XX.XXX.XXX.XX    10.240.0.21    i-08ba89f08d6e97225
worker-1
XX.XX.XX.XXX     10.240.0.10    i-0e77888e5509c86c7
controller-0
XX.XXX.XX.XXX    10.240.0.12    i-0066a7c99ae9869b3
controller-2
XX.XXX.XX.XXX    10.240.0.20    i-01ffaf1b1258b341b
worker-0
XX.XXX.XX.XXX    10.240.0.22    i-03aa4e84d88ef29e5
worker-2
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances. Save the IP addresses in 

```bash
for i in 0 1 2; do
  declare ip_worker_${i}=`aws ec2 describe-instances --filter "Name=vpc-id,Values=$vpcId" "Name=tag:Name,Values=worker-${i}" --query 'Reservations[].Instances[].PublicIpAddress' --output text`
  declare ip_controller_${i}=`aws ec2 describe-instances --filter "Name=vpc-id,Values=$vpcId" "Name=tag:Name,Values=controller-${i}" --query 'Reservations[].Instances[].PublicIpAddress' --output text`
done
```

Test SSH access to the `controller-0` compute instances:

```bash
ssh -l ubuntu $ip_controller_0
```

After this you'll be logged into the `controller-0` instance:

```bash
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1011-aws x86_64)

...

Last login: Thu Jul  5 19:50:41 2018 from XXX.XX.XXX.XX
```

Populate hosts file and set the hostname.

```bash
sudo /bin/bash -c "echo -e \"\n10.240.0.10 controller-0\n10.240.0.11 controller-1\n10.240.0.12 controller-2\n10.240.0.20 worker-0\n10.240.0.21 worker-1\n10.240.0.22 worker-2\" >> /etc/hosts"
sudo /bin/bash -c "grep `curl -s http://169.254.169.254/latest/meta-data/local-ipv4` /etc/hosts |cut -d ' ' -f 2 > /etc/hostname"
sudo /bin/bash -c "hostname `grep $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4) /etc/hosts |cut -d ' ' -f 2`"
```

Type `exit` at the prompt to exit the `controller-0` compute instance and repeat for controller-1, controller-2, worker-0, worker-1 and worker-2.

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
