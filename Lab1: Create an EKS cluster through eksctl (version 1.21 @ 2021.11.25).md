## Lab1: Create an EKS cluster through eksctl (version 1.21 @ 2021.11.25)
#### !Tips, all commands here are tested on AMZN Linux 2

#### If you don't have any AMAZ linux 2-based workstation for your lab, please create it via following link
[http://sharewithotheropen.s3-website.cn-northwest-1.amazonaws.com.cn](http://sharewithotheropen.s3-website.cn-northwest-1.amazonaws.com.cn)


## 1.Install needed tools on workstation: awscli v2, kubectl, eksctl, aws-iam-authenticator, kubectx and kubens
#### 1) Inastall and configure awscli v2
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip

[new install]
sudo ./aws/install

[update install] If found preexisting AWS CLI installation, please execute following command to install
sudo ./aws/install --update


complete -C '/usr/local/bin/aws_completer' aws; echo "complete -C '/usr/local/bin/aws_completer' aws" >> ~/.bashrc
source ~/.bash_profile
```
Check awscli version, 
```
aws --version
```
If the version of awscli is older(v1), you can follow the official link below to update awscli v2:  
[https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade)  

configure credential file:
```
[ec2-user@ip-172-31-1-111 ~]$ aws configure
AWS Access Key ID [None]:  Input your AK 
AWS Secret Access Key [None]:  Input your SK
Default region name [cn-northwest-1]: By default 宁夏 region
Default output format [json]:  
```
Official link for configure aws cli:  
[https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html](https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html)

#### 2)Install kubectl(Kubernetes 1.21)
```
curl -o kubectl https://amazon-eks.s3.cn-north-1.amazonaws.com.cn/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin

source <(kubectl completion bash); echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bash_profile  
```

Check kubectl version, 
```
kubectl version --short
```
Official link for install kubectl(aws):  
[https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-kubectl.html](https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-kubectl.html)  
Official link for install kubectl(Kubernetes):  
[https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

#### 3)Install eksctl
```
curl -OL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" 
#curl -OL https://sharewithotheropen.s3.cn-northwest-1.amazonaws.com.cn/eksctl_Linux_amd64.tar.gz
tar xvf eksctl_*_amd64.tar.gz
sudo mv ./eksctl /usr/local/bin

. <(eksctl completion bash); echo ". <(eksctl completion bash)" >> ~/.bashrc
source ~/.bash_profile
```
Check eksctl version, 
```
eksctl version
```
Official link for install eksctl:  
[https://docs.amazonaws.cn/en_us/eks/latest/userguide/eksctl.html](https://docs.amazonaws.cn/en_us/eks/latest/userguide/eksctl.html)  

#### 4)Install aws-iam-authenticator
```
curl -OL https://amazon-eks.s3.cn-north-1.amazonaws.com.cn/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
chmod +x ./aws-iam-authenticator
sudo mv ./aws-iam-authenticator /usr/local/bin
```
Official link for install aws-iam-authenticator:  
[https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-aws-iam-authenticator.html](https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-aws-iam-authenticator.html) 

#### 5)[optional] Install kubectx and kubens      
``` 
curl -OL https://github.com/ahmetb/kubectx/releases/download/v0.9.3/kubectx_v0.9.3_linux_x86_64.tar.gz
#curl -OL https://sharewithotheropen.s3.cn-northwest-1.amazonaws.com.cn/kubectx_v0.9.1_linux_x86_64.tar.gz
tar zxvf kubectx_v*_linux_x86_64.tar.gz
chmod +x ./kubectx
sudo mv ./kubectx /usr/local/bin

curl -OL https://github.com/ahmetb/kubectx/releases/download/v0.9.3/kubens_v0.9.3_linux_x86_64.tar.gz
tar zxvf kubens_v*_linux_x86_64.tar.gz
chmod +x ./kubens
sudo mv ./kubens /usr/local/bin
```
Official link for download:  
[https://github.com/ahmetb/kubectx/releases](https://github.com/ahmetb/kubectx/releases)

####  6)[optional] Install jq
``` 
sudo yum install -y jq
``` 

####  7)Create an SSH key pair,replace your-key-name to what you like, such as: jerrykey 
``` 
aws ec2 create-key-pair --key-name your-key-name --query 'KeyMaterial' --output text > your-key-name.pem  
chmod 400 your-key-name.pem
``` 
Example:
``` 
[ec2-user@ip-172-31-1-73 ~]$ aws ec2 create-key-pair --key-name jerrykey --query 'KeyMaterial' --output text > jerrykey.pem  
[ec2-user@ip-172-31-1-73 ~]$ ls -la | grep jerrykey
-rw-rw-r--  1 ec2-user ec2-user      1679 Nov 25 14:20 jerrykey.pem
[ec2-user@ip-172-31-1-73 ~]$ chmod 400 jerrykey.pem 
[ec2-user@ip-172-31-1-73 ~]$ ls -la | grep jerrykey
-r--------  1 ec2-user ec2-user      1679 Nov 25 14:20 jerrykey.pem
```  

##  2. Create EKS cluster on AWS China region through eksctl, we choose Ningxia region here. You have two ways to create the EKS cluster, one is single command line, the other is with yaml file 
#### 1) one is single command line
```
# 环境变量
# AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
# CLUSTER_NAME 集群名称
# 替换ekswork-yourname为适合值，比如eksworkjerry
export CLUSTER_NAME=ekswork-yourname
# 替换ng-ekswork-yourname为适合值，比如ng-eksworkjerry
export NODEGROUP_NAME=ng-ekswork-yourname
export INSTANCE_TYPE=t3.medium
# 替换your-key为你的key的名称
export AWS_KEY=your-key-name
# 替换1.19为你需要的版本
export VERSION=1.21
```
Create eks cluster through follow single command line
```
eksctl create cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --nodegroup-name ${NODEGROUP_NAME} --node-type ${INSTANCE_TYPE} --nodes 3 --nodes-min 1 --nodes-max 4 --ssh-access --ssh-public-key ${AWS_KEY} --managed --version ${VERSION}
```

#### 2) Prepare EKS cluster yaml file - eksworkjerry.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkjerry
  region: cn-northwest-1
  version: "1.21"

managedNodeGroups:
  - name: ng-eksworkjerry-01
    instanceType: t3.medium
    instanceName: ng-eksworkjerry
    desiredCapacity: 3
    minSize: 1
    maxSize: 4 
    volumeSize: 100
    ssh:
      publicKeyName: <your-key-name>
      allow: true
      enableSsm: true 
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true 
```

Create eks cluster through follow single command line
```
eksctl create cluster -f eksworkjerry.yaml
```

#### 3) Prepare EKS cluster yaml file for existing VPC with private network only - eksworkjerryvpc.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkjerryvpc
  region: cn-northwest-1
  version: "1.21"

vpc:
#  This vpc is with 3 subnets, nat gateway is within 1 public subnet 
#  Those private subnets are with route table which contains route entry to nat gateway
#  id: "vpc-0c445fc9cb3c357c8"  # (optional, must match VPC ID used for each subnet below)
#  cidr: "10.62.0.0/16"       # (optional, must match CIDR used by the given VPC)
  subnets:
    # must provide 'private' and/or 'public' subnets by availibility zone as shown
    private:
      cn-northwest-1a:
        id: "subnet-0dbdffd43d569a3a1"
#        cidr: "10.62.2.0/24" # (optional, must match CIDR used by the given subnet)

      cn-northwest-1b:
        id: "subnet-0e2355b831c9b187e"
#        cidr: "10.62.3.0/24"  # (optional, must match CIDR used by the given subnet)

managedNodeGroups:
  - name: ng-eksworkjerryvpc-01
    instanceType: t3.medium
    instanceName: ng-eksworkjerryvpc
    desiredCapacity: 3
    minSize: 1
    maxSize: 4 
    volumeSize: 100
    privateNetworking: true # if only 'Private' subnets are given, this must be enabled
    ssh:
      publicKeyName: jerrykey
      allow: true
      enableSsm: true 
```

Create eks cluster through follow single command line
```
eksctl create cluster -f eksworkjerryvpc.yaml
```


#### 4) There are several yaml examples for creating an EKS cluster under different scenarios
[https://github.com/weaveworks/eksctl/tree/main/examples](https://github.com/weaveworks/eksctl/tree/main/examples)
