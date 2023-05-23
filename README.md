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


- Provisioning Client/Server certificates for all the components that will communicate with the api-server by using the root CA to request more certificates which the different Kubernetes components, i.e. clients and server, will use to have encrypted communication.

- Generating the Certificate Signing Request (CSR), Private Key and the Certificate for the Kubernetes Master Nodes (api-server).
```
{
cat > master-kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
   "hosts": [
   "127.0.0.1",
   "172.31.0.10",
   "172.31.0.11",
   "172.31.0.12",
   "ip-172-31-0-10",
   "ip-172-31-0-11",
   "ip-172-31-0-12",
   "ip-172-31-0-10.${AWS_REGION}.compute.internal",
   "ip-172-31-0-11.${AWS_REGION}.compute.internal",
   "ip-172-31-0-12.${AWS_REGION}.compute.internal",
   "${KUBERNETES_PUBLIC_ADDRESS}",
   "kubernetes",
   "kubernetes.default",
   "kubernetes.default.svc",
   "kubernetes.default.svc.cluster",
   "kubernetes.default.svc.cluster.local"
  ],
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  master-kubernetes-csr.json | cfssljson -bare master-kubernetes
}
```

![ti18](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/d8887c0b-e58b-41a6-9f13-6b863d53da59)

- Generating Client Certificate and Private Key for **kube-scheduler**
```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-scheduler",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```


![ti18](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/59e255e8-3756-491b-bdd3-08b89742cad1)

![ti19](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/4ce8e9e1-3989-4697-8c1b-ae61c60c1d17)


- Generating Client Certificate and Private Key for **kube-proxy**
```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:node-proxier",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

![ti20](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/904078be-ce68-4b64-829e-3a38138f0c76)

- Generating Client Certificate and Private Key for **kube-controller-manager**
```
{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:kube-controller-manager",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

![ti21](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/8e19ecc6-6532-4dc6-bdab-0f6fafd3b856)


- Generating Client Certificate and Private Key for **kubelet**
```
for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  instance_hostname="ip-172-31-0-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:nodes",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    ${NAME}-worker-${i}-csr.json | cfssljson -bare ${NAME}-worker-${i}
done
```

![ti22](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/7184888e-aa82-457a-b04c-e840f79982d5)

- Generating Client Certificate and Private Key for **kubernetes admin user**
```
{
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "UK",
      "L": "England",
      "O": "system:masters",
      "OU": "DAREY.IO DEVOPS",
      "ST": "London"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}
```

![ti23](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/3ca5ebdd-7455-47b5-8e09-e9df31034721)


- Generating Client Certificate and Private Key for **Token Controller**

It is a part of the Kubernetes Controller Manager responsible for generating and signing service account tokens which are used by pods or other resources to establish connectivity to the api-server
```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
}
```

![ti24](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/39c51341-6e31-4107-9e51-a331e829d238)



## STEP 6: Distributing The Client And Server Certificates
- Sending all the client and server certificates to their respective instances. Starting from **worker nodes**
```
for i in 0 1 2; do
  instance="${NAME}-worker-${i}"
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/; \
done
```

- For the **master nodes**:
```
for i in 0 1 2; do
instance="${NAME}-master-${i}" \
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  scp -i ../ssh/${NAME}.id_rsa \
    ca.pem ca-key.pem service-account-key.pem service-account.pem \
    master-kubernetes.pem master-kubernetes-key.pem ubuntu@${external_ip}:~/;
done
```


To streamline the process and ensure smooth operation when the Kubernetes cluster is up and running, the next step involves creating kubeconfig files. These files are essential for Kubernetes clients to locate and authenticate with the Kubernetes API Servers.

To accomplish this, the following environment variables will be set up for use in multiple commands, saving time and effort:

Client tool: The first requirement is to have the kubectl client tool installed. This tool is widely used for interacting with Kubernetes clusters.

Generating kubeconfig files: Kubeconfig files need to be generated for various clients, including kubelet, controller manager, kube-proxy, scheduler, and the admin user. These files will contain the necessary configuration details and credentials required for authentication and communication with the Kubernetes API Servers.

By creating and properly configuring the kubeconfig files, managing the Kubernetes cluster becomes more convenient and efficient. The `kubectl` command line tool is essential for executing commands and managing the cluster.

- Creating an environment variables for reuse by multiple commands:`$ KUBERNETES_API_SERVER_ADDRESS=$(aws elbv2 describe-load-balancers --load-balancer-arns ${LOAD_BALANCER_ARN} --output text --query 'LoadBalancers[].DNSName')`
- Generating the kubelet kubeconfig file:
```
for i in 0 1 2; do

instance="${NAME}-worker-${i}"
instance_hostname="ip-172-31-0-2${i}"

 # Set the kubernetes cluster in the kubeconfig file
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://$KUBERNETES_API_SERVER_ADDRESS:6443 \
    --kubeconfig=${instance}.kubeconfig

# Set the cluster credentials in the kubeconfig file
  kubectl config set-credentials system:node:${instance_hostname} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

# Set the context in the kubeconfig file
  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:node:${instance_hostname} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

- Generating the kube-proxy kubeconfig
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

![ti29](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/3afc74fc-a785-47cf-89d7-4246bf29ced3)

- Generating the Kube-Controller-Manager kubeconfig:
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

![ti30](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/dcddb3f0-274e-46eb-933d-6de51725da64)


- Generating the Kube-Scheduler Kubeconfig:
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

![ti31](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/82d1edf0-7570-4f98-9eec-6ec0d7879a31)


- Generating the kubeconfig file for the admin user
```
{
  kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=${NAME} \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

![ti32](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/3930b1d7-7de0-4550-94a9-cff64b62ef3d)


- Distributing the files to thier respective servers using scp and for loop:

**For Master nodes**

![ti34](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/388ab767-9aa6-4a31-8752-4a52024345e3)


**For Worker nodes**

![ti33](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/d3c78c1f-ec8e-4516-9fae-1a978d203da2)



To address the security risk associated with Kubernetes' use of etcd, a distributed key-value store for storing cluster state, application configurations, and secrets, it is crucial to take measures to encrypt the data at rest. By default, the data stored in etcd is in plain text, making it vulnerable to exploitation if an attacker gains access to the database.

In order to mitigate this risk, it is necessary to prepare and implement encryption for etcd data at rest. Encrypting etcd ensures that even if the data is accessed by unauthorized entities, it remains unreadable and protected. By encrypting the data, the confidentiality and integrity of the stored information are maintained, adding an additional layer of security to the Kubernetes cluster.

By taking the necessary steps to encrypt etcd at rest, the overall security posture of the Kubernetes cluster is significantly improved, reducing the risk of data compromise and unauthorized access.

 - Generating encryption key and encoding it using base64:`ETCD_ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)`
 - Creating an encryption-config.yaml file
 ```
 cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ETCD_ENCRYPTION_KEY}
      - identity: {}
EOF
```

SSH into the controller server

**For Master node 1**
```
master_1_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-0" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_1_ip}
```

**For Master node 2**
```
master_2_ip=$(
aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-1" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_2_ip}
```

**For Master node 3**
```
master_3_ip=$(aws ec2 describe-instances \
--filters "Name=tag:Name,Values=${NAME}-master-2" \
--output text --query 'Reservations[].Instances[].PublicIpAddress')
ssh -i k8s-cluster-from-ground-up.id_rsa ubuntu@${master_3_ip}
```

- Downloading and installing **etcd**
```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```
- Extracting and installing the etcd server and the etcdctl command line utility:
```
{
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}
```

- Configuring the etcd server
```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem master-kubernetes-key.pem master-kubernetes.pem /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance

``
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)

echo ${ETCD_NAME}
```

- Creating the **etcd.service** systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-0=https://172.31.0.10:2380,master-1=https://172.31.0.11:2380,master-2=https://172.31.0.12:2380 \\
  --cert-file=/etc/etcd/master-kubernetes.pem \\
  --key-file=/etc/etcd/master-kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/master-kubernetes.pem \\
  --peer-key-file=/etc/etcd/master-kubernetes-key.pem \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Starting and enabling the etcd Server
```
{
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
}
```

- Verifying the etcd installation
```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/master-kubernetes.pem \
  --key=/etc/etcd/master-kubernetes-key.pem
```

![ti35](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/31fbe96c-0dc2-4b3a-8094-01230d6921a6)


- Checking the etcd service status:`$ systemctl status etcd`


![ti40](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/f8b1b17e-5909-495e-85dd-32b5073437bf)


## STEP 9: Configuring The Components For The Control Plane On The Master/Controller Nodes

- Creating the Kubernetes configuration directory:`$ sudo mkdir -p /etc/kubernetes/config`
- Downloading the official Kubernetes release binaries:
```
wget -q --show-progress --https-only --timestamping \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
"https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```

- Installing the Kubernetes binaries:
```
{
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```
- Configuring the Kubernetes API Server:
```
{
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem master-kubernetes-key.pem master-kubernetes.pem \
service-account-key.pem service-account.pem \
encryption-config.yaml /var/lib/kubernetes/
}
```

- The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieving the internal IP address for the current compute instance:`$ export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)`
- Creating the kube-apiserver.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/master-kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/master-kubernetes-key.pem\\
  --etcd-servers=https://172.31.0.10:2379,https://172.31.0.11:2379,https://172.31.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/master-kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/master-kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${INTERNAL_IP}:6443 \\
  --service-cluster-ip-range=172.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/master-kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/master-kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Moving the kube-controller-manager kubeconfig into place:`$ sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/`
- Exporting some variables to retrieve the vpc_cidr which will be required for the bind-address flag:
```
export AWS_METADATA="http://169.254.169.254/latest/meta-data"
export EC2_MAC_ADDRESS=$(curl -s $AWS_METADATA/network/interfaces/macs/ | head -n1 | tr -d '/')
export VPC_CIDR=$(curl -s $AWS_METADATA/network/interfaces/macs/$EC2_MAC_ADDRESS/vpc-ipv4-cidr-block/)
export NAME=k8s-cluster-from-ground-up
```

- Creating the kube-controller-manager.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=${VPC_CIDR} \\
  --cluster-name=${NAME} \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=172.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Moving the kube-scheduler kubeconfig into place:
```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
sudo mkdir -p /etc/kubernetes/config
```
- Creating the kube-scheduler.yaml configuration file:
```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

- Creating the kube-scheduler.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Starting the Controller Services
```
{
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

![ti41](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/268c1bc5-1cae-438a-bf71-be1791c6055e)


## STEP 10: Testing that Everything is working fine
- To get the cluster details run:`$ kubectl cluster-info  --kubeconfig admin.kubeconfig`
- To get the current namespaces:`$ kubectl get namespaces --kubeconfig admin.kubeconfig`


![ti44](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/6337776a-ffe4-41ee-a21c-8efd3bb1cef8)

![ti45](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/c452843b-5d23-457a-b02f-2f04e5e7eef3)

![ti46](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/6d659c34-fd60-4f29-8df2-dadb7161e674)

- To reach the Kubernetes API Server publicly:`$ curl --cacert /var/lib/kubernetes/ca.pem https://$INTERNAL_IP:6443/version`

- To get the status of each component:`$ kubectl get componentstatuses --kubeconfig admin.kubeconfig`

![ti47](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/539b5cad-0c29-4937-ba03-c5e79f0c5c12)

![ti48](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/ccf78ecb-9795-4188-acbd-760ed9814812)

![ti49](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/81919ec9-8149-4c06-91c5-55a1ccef0768)


## STEP 11: Configuring Role Based Access Control
- Configuring Role Based Access Control (RBAC) on one of the controller(master) nodes so that the api-server has necessary authorization for for the kubelet.

Creating the **ClusterRole**
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

- Creating the **ClusterRoleBinding** to bind the kubernetes user with the role created above
```
cat <<EOF | kubectl --kubeconfig admin.kubeconfig  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

- The **RBAC** permissions is configured to allow the Kubernetes API Server to access the Kubelet API on each worker nodes. Creating the system:kube-apiserver-to-kubelet ClusterRole with permissions to access the Kubelet API and perform most common tasks associated with managing pods on the worker nodes:
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

![ti50](https://github.com/busolagbadero/Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes./assets/94229949/5a3d30a7-baaf-424b-883d-67e89cb8e2b8)


- Binding the system:kube-apiserver-to-kubelet ClusterRole to the kubernetes user so that API server can authenticate successfully to the kubelets on the worker nodes:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

