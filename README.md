## Kubernetes setup on AWS

https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md

### 1. Setup Kops and Aws CLI
```
brew update install kops
```

### 2. Create aws access key

https://console.aws.amazon.com/iam/home?region=ap-south-1#/security_credentials$access_key
```
pip3 install awscli --upgrade --user
aws configure
## If aws command doesn't work check the $PATH variable in ~/.bash_profile:
## python bin must be added.
## Enter access key, secret, region, and json as format.
```

### 3. Create Cluster


#### 3.1 Create bucket for kops state storage. 
```
aws s3 mb s3://cluster.dev.niteshagarwal.in
```

#### 3.2 ENV variables for KOPS
```
export KOPS_STATE_STORE=s3://cluster.dev.niteshagarwal.in
export ROUTE53_ZONE=ap-south-1a
export AMI_ID=ami-0d758c1134823146a
export CLUSTER_NAME=dev.niteshagarwal.in

## To get zone use zone name from the following cmd output
aws ec2 describe-availability-zones --region ap-south-1

## To get AMI complete path like 099720109477/ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-20210223 use ImageLocation
aws ec2 describe-images --region ap-south-1 --image-id ${AMI_ID}

## Update AMI_ID
export AMI_ID=099720109477/ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-20210223

```

#### 3.3 DNS
To configure DNS for domains created outside aws- DNS delegation is required.
Create another Hosted Zone (public or private) and create NS record in the parent with this hosted zone.
```
export ROUTE53_ZONE=Z0808*******SX6F7N
dig ns dev.niteshagarwal.in
```

#### 3.4 KEY_PAIR
To configure Key Pair go to aws ec2 console and create key pair. Using .cer file generate public key.
```
chmod 400 dev-niteshagarwal-in.cer
ssh-keygen -y -f dev-niteshagarwal-in.cer > key.pub
chmod 400 key.pub
```

#### 3.5 Create Cluster

```
kops create cluster \
  --cloud aws \
  --dns public \
  --dns-zone ${ROUTE53_ZONE} \
  --topology private \
  --networking weave \
  --associate-public-ip=false \
  --encrypt-etcd-storage \
  --network-cidr 10.2.0.0/16 \
  --image ${AMI_ID} \
  --kubernetes-version 1.10.11 \
  --master-size t2.medium \
  --master-count 1 \
  --master-zones ap-south-1a \
  --master-volume-size 8 \
  --zones ap-south-1a \
  --node-size t2.medium \
  --node-count 2 \
  --node-volume-size 12 \
  --ssh-public-key=$PWD/key.pub \
  ${CLUSTER_NAME}

kops create secret sshpublickey admin -i $PWD/key.pub --name ${CLUSTER_NAME}


## Upgrade cluster
kops upgrade cluster ${CLUSTER_NAME} --yes

## Update Cluster
kops update cluster ${CLUSTER_NAME}
kops update cluster ${CLUSTER_NAME} --yes
```

#### 3.6 Validate Cluster
```
kops export kubecfg --admin
kops validate cluster --wait 10m

## list nodes: 
kubectl get nodes --show-labels

## ssh to the master
ssh -i ~/.ssh/id_rsa ubuntu@api.dev.niteshagarwal.in

kops export kubecfg --admin
```

#### 3.7 Add Bastion Node
```
kops create instancegroup bastions --role Bastion --subnet utility-ap-south-1a --name ${CLUSTER_NAME}
#<var/folders/k6/81**sc/T/kops-edit-fpllnyaml
kops export kubecfg --admin

aws elb --output=table describe-load-balancers|grep DNSName.\*bastion|awk '{print $4}'
export bastion_elb_url=<Output of above>
ssh-add -K dev-niteshagarwal-in.cer
ssh -A ubuntu@${bastion_elb_url}
```

#### 3.8 Delete Cluster
```
kops delete cluster ${CLUSTER_NAME} --yes
```

### 4. Manage Cluster

```
kops get ig
```

#### 4.1 Shutdown cluster without terminating

```
kops edit ig nodes-ap-south-1a
## Set min and max to 0

kops edit ig master-ap-south-1a
## Set min and max to 0

kops edit ig bastions
## Set min and max to 0

kops update cluster ${CLUSTER_NAME} --yes
kops rolling-update cluster ${CLUSTER_NAME} --yes
```
