# Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes.

# INTRODUCTION

The goal of this project is to gain a deeper understanding of the various aspects involved in setting up a Kubernetes cluster by manually configuring each component without relying on any automated tools or helpers. The following steps outline the process of manually installing and setting up the Kubernetes cluster from scratch.

## STEP 1: Installing Kubectl On The Local Machine(Linux)

- Downloading the binary: `$ wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl`
- Making it executable:`$ chmod +x kubectl`
- Moving the file to the Bin directory:`$ sudo mv kubectl /usr/local/bin/`
- Verifying that kubectl version 1.21.0 or higher is installed:`$ kubectl version --client`


## STEP 2: Installing CFSSL And CFSSLJSON

CFSSL, developed by Cloudflare, is an open-source tool utilized for establishing a Public Key Infrastructure (PKI) that enables the generation, signing, and consolidation of TLS certificates.

- Downloading the binary:
``` 
$ wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
```

- Making it executable: `$ chmod +x cfssl cfssljson`
- Moving the file to the bin directory:`$ sudo mv cfssl cfssljson /usr/local/bin/`


## STEP 3: Configuring The Network Infrastructure
- Creating a directory named k8s-cluster-from-ground-up and changing directory: `$ mkdir k8s-cluster-from-ground-up && cd k8s-cluster-from-ground-up`
- Creating a VPC and storing its ID as a variable: 
```
$ VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)
```
- Tagging the VPC so that it is named:
```
$ NAME=k8s-cluster-from-ground-up

$ aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}
```
- Enabling DNS support for the VPC:
```
$ aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-support '{"Value": true}'
```
- Enabling DNS support for hostnames:
```
$ aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
```
- Setting the required region:`$ AWS_REGION=eu-central-1`
- Configuring DHCP Options Set:
```
$ DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')
```
- Tagging the DHCP Option set:
```
$ aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=${NAME}
```
- Associating the DHCP Option set with the VPC:
```
$ aws ec2 associate-dhcp-options \
  --dhcp-options-id ${DHCP_OPTION_SET_ID} \
  --vpc-id ${VPC_ID}
```

![ti1](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/d3b97a9d-1bc5-4bd2-bc8a-ed6fe44c66ba)

![ti2](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/d8bab463-abc0-4867-9b8a-5a2675338c5f)


![ti3](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/1f2773d5-5f3f-4212-bb0c-238175026cea)


![ti4](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/4e646953-60ae-4827-9064-c6ff53d1d50e)


- Creating the Subnet: 
```
$ SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')
```
- Tagging the Subnet:
```
$ aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}
```
- Creating the Internet Gateway and tagging it:
```
$ INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')

$ aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${NAME}
```
- Attaching the Internet Gateway to the VPC:
```
$ aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
```
- Create route tables, associate the route table to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway:
```
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}
aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
```

![ti5](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/ca09906d-eb4c-4d8f-a090-ee264d82bf96)

![ti6](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/3dd2562f-caaa-437b-9996-03cab1850cbe)

![ti7](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/c57abaa9-4226-476f-8d32-93af3c75fecb)


![ti8](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/afcfa7dd-3fef-4b72-a4bd-b54dc06283d3)


![ti9](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/227ff5bf-7b66-4740-93c6-417c6ff0a361)


- Configuring Security Groups
```
# Create the security group and store its ID in a variable
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name ${NAME} \
  --description "Kubernetes cluster security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

# Create the NAME tag for the security group
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=${NAME}

# Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'

# # Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'
# Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0

# Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Create ICMP ingress for all types
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0
  ```
  
  ![ti10](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/20d99b68-b789-401a-9733-3bf109a67844)


- Creating a network Load balancer:
```
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
  --name ${NAME} \
  --subnets ${SUBNET_ID} \
  --scheme internet-facing \
  --type network \
  --output text --query 'LoadBalancers[].LoadBalancerArn')
```
- Creating a target group for it
```
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name ${NAME} \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
```
- Registering targets - though there are no real targets but the private IP addresses are selected so that when the nodes become available they will be used as targets:
```
aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=172.31.0.1{0,1,2}
```
- Creating a listener to listen for requests and forward to the target nodes on TCP port 6443
```
aws elbv2 create-listener \
--load-balancer-arn ${LOAD_BALANCER_ARN} \
--protocol TCP \
--port 6443 \
--default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
--output text --query 'Listeners[].ListenerArn'
```
- Retrieving the Kubernetes Public address and storing it 
```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')
```

![ti11](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/c3279227-4aba-4138-b9ea-bf8a40a9267b)

![ti12](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/248c4364-ab2a-4992-92d3-d92ff7ac2879)

![ti13](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/ef3a4bde-6e88-4157-88f0-60b2477f103d)

![ti14](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/0485730f-2474-4c24-99b5-dd66e0f594e3)

## STEP 4: Creating Compute Resources

- Retrieving an image ID to create EC2 instances:
```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```
- Creating an SSH Key-Pair:
```
$ mkdir -p ssh

$ aws ec2 create-key-pair \
  --key-name ${NAME} \
  --output text --query 'KeyMaterial' \
  > ssh/${NAME}.id_rsa

$ chmod 600 ssh/${NAME}.id_rsa
```
- Creating 3 Master nodes:
```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.1${i} \
    --user-data "name=master-${i}" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-master-${i}"
done
```

- Creating 3 worker nodes:
```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name ${NAME} \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 172.31.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=172.20.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=${NAME}-worker-${i}"
done
```

![ti15](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/ac370fbb-3093-44e2-9480-a4167bb5559a)

![ti16](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/a98a7404-6187-4bba-a043-dcdb9ac0d910)

## STEP 5: Preparing The Self-Signed Certificate Authority And Generating The TLS Certificates

 The PKI Infrastructure is provisioned using **cfssl** which will have a Certificate Authority which will then generate certificates for all the individual components:
 - Creating a directory called **ca-authority** and cd into it:`$ mkdir ca-authority && cd ca-authority`
- Generating the CA configuration file, Root Certificate, and Private key:
```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "Kubernetes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

![ti17](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/49484d09-eeec-427c-9d59-eef6bfa99a14)




