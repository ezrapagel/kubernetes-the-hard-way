# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

---

- GCP

```bash
gcloud -q compute instances delete \
  controller-0 controller-1 controller-2 \
  worker-0 worker-1 worker-2
```

- AWS

```bash
vpcId=`aws ec2 describe-vpcs \
  --filters Name=tag:Name,Values=Kubernetes \
  --query Vpcs[0].VpcId \
  --output text`
```

```bash
aws ec2 terminate-instances --instance-ids `aws ec2 describe-instances \
  --filter Name=vpc-id,Values=$vpcId \
  --query Reservations[*].Instances[0].InstanceId \
  --output text`
```

---

## Networking

Delete the external load balancer network resources:

---

- GCP

```bash
 gcloud -q compute forwarding-rules delete kubernetes-forwarding-rule \
  --region $(gcloud config get-value compute/region)

gcloud -q compute target-pools delete kubernetes-target-pool

gcloud -q compute http-health-checks delete kubernetes

gcloud -q compute addresses delete kubernetes-the-hard-way
```

- AWS

```bash
aws elb delete-load-balancer --load-balancer-name kubernetes
```

---

Delete the `kubernetes-the-hard-way` firewall rules:

---

- GCP

```bash
gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external \
  kubernetes-the-hard-way-allow-health-check
```

- AWS

```bash
aws ec2 delete-security-group --group-id `aws ec2 describe-security-groups \
  --filter Name=vpc-id,Values=$vpcId Name=group-name,Values=kubernetes* \
  --query SecurityGroups[*].GroupId \
  --output text`
```

---

Delete the `kubernetes-the-hard-way` network VPC:

---

- GCP

```bash
gcloud -q compute routes delete \
  kubernetes-route-10-200-0-0-24 \
  kubernetes-route-10-200-1-0-24 \
  kubernetes-route-10-200-2-0-24

gcloud -q compute networks subnets delete kubernetes

gcloud -q compute networks delete kubernetes-the-hard-way
```

- AWS

```bash
subnetId=`aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=Kubernetes-Subnetwork \
  --query Subnets[0].SubnetId \
  --output text`
aws ec2 delete-subnet --subnet-id $subnetId
```

```bash
aws ec2 delete-vpc --vpc-id=$vpcId
```

---
