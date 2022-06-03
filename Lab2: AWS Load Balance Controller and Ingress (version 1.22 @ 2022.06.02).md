## Lab2: AWS Load Balance Controller and Ingress (version 1.22 @ 2022.06.02)
#### !Tips, all commands here are tested on AMZN Linux 2   
#### It has been tested on EKS 1.22

## 1.Prepare EKS cluster yaml file - eksgo04-cluster.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksgo04
  region: cn-northwest-1
  version: "1.19"

managedNodeGroups:
  - name: ng-eksgo04-01
    instanceType: t3.xlarge
    instanceName: ng-eksgo04
    desiredCapacity: 3
    minSize: 1
    maxSize: 8
    volumeSize: 100
    ssh:
      publicKeyName: key-ningxia-new
      allow: true
      enableSsm: true
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true
```

## 2.Create EKS cluster
```
eksctl create cluster -f  eksgo04-cluster.yaml
```

## 3.Set parameters and get info
```
export CLUSTER_NAME=eksgo04
export AWS_REGION=cn-northwest-1
```

Get vpc-id and subnet-id
```
[ec2-user@ip-172-31-1-111 ~]$ aws eks describe-cluster --name ${CLUSTER_NAME} | jq .cluster.resourcesVpcConfig.subnetIds,.cluster.resourcesVpcConfig.vpcId
[
  "subnet-027e04820f7bb116a",
  "subnet-010b1123cae0aa3a7",
  "subnet-0d388bb9d77d4fa8a",
  "subnet-0aecc3d7f8bae3150",
  "subnet-072caddf70318fc78",
  "subnet-0419f21a3034adedf"
]
"vpc-0454c48d0fcd3254c”
```

Check Tags related to subnets
```
[ec2-user@ip-172-31-1-111 ~]$ aws ec2 describe-subnets --filters "Name=subnet-id,Values="*subnet-027e04820f7bb116a*"" | jq .Subnets[0].Tags

[
  {
    "Key": "aws:cloudformation:stack-id",

    "Value": "arn:aws-cn:cloudformation:cn-northwest-1:<Your_Account_ID>:stack/eksctl-eksgo04-cluster/a052c650-88cb-11eb-8d28-06fd69f21d7a"

  },
  {
    "Key": "kubernetes.io/cluster/eksgo04",
    "Value": "shared"
  },
  {
    "Key": "Name",
    "Value": "eksctl-eksgo04-cluster/SubnetPublicCNNORTHWEST1C"
  },
  {
    "Key": "aws:cloudformation:logical-id",
    "Value": "SubnetPublicCNNORTHWEST1C"
  },
  {
    "Key": "eksctl.cluster.k8s.io/v1alpha1/cluster-name",
    "Value": "eksgo04"
  },
  {
    "Key": "aws:cloudformation:stack-name",
    "Value": "eksctl-eksgo04-cluster"
  },
  {
    "Key": "alpha.eksctl.io/eksctl-version",
    "Value": "0.40.0"
  },
  {
    "Key": "alpha.eksctl.io/cluster-name",
    "Value": "eksgo04"
  },
  {
    "Key": "kubernetes.io/role/elb",
    "Value": "1"
  }
]
```

Set every subnets in your VPC for EKS cluster to have tag

```
for NAME in $(aws eks describe-cluster --name ${CLUSTER_NAME} | jq .cluster.resourcesVpcConfig.subnetIds[])
do
 eval aws ec2 create-tags --resources ${NAME} --tags Key="kubernetes.io/cluster/${CLUSTER_NAME}",Value=shared
done
```

## 4.To allow the cluster to use AWS Identity and Access Management (IAM) for service accounts, create IAM OIDC provider

View your cluster's **OpenID Connect provider URL**(OIDC endpoint/URL is used to create an IAM OIDC identity provider for EKS cluster, determining the location of the OpenID Provider )
```
aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text
```
Example output
https://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

List the IAM OIDC identity providers in your account.

```
aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | awk -F/ '{print $NF}')
```

Example output
```
"Arn": "arn:aws:iam::111122223333:oidc-provider/[oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E](http://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E)"
```

If the there is no IAM OIDC identity provider, you can create an IAM OIDC identity provider ( Service Provider ) for your cluster with the following command.

```
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}

2021-03-19 09:28:49 [ℹ]  will create IAM Open ID Connect provider for cluster "eksgo04" in "cn-northwest-1"
2021-03-19 09:28:49 [✔]  created IAM Open ID Connect provider for cluster "eksgo04" in "cn-northwest-1”
```

Check IAM OIDC identity provider again

```
[ec2-user@ip-172-31-1-111 ~]$ aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | awk -F/ '{print $NF}')
            "Arn": "arn:aws-cn:iam::111122223333:oidc-provider/oidc.eks.cn-northwest-1.amazonaws.com.cn/id/EB8440E65FAFC194F7021319931A59AD"
```

## 5.Create an IAM policy for the service account using the correct permissions 

下载IAM policy file
```
curl -OL https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/install/iam_policy_cn.json
```
下载cert-manager yaml file   
cert-manager官方guide: [https://cert-manager.io/docs/installation/](https://cert-manager.io/docs/installation/)
```
curl -OL https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```
下载AWS Load Balance Controller yaml file
```
curl -OL https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.1/v2_4_1_full.yaml
```

创建名为AWSLoadBalancerControllerIAMPolicy的IAM policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json
```
记录返回的Plociy ARN
```
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
```

## 6.To create a service account
```
eksctl create iamserviceaccount \
       --cluster=${CLUSTER_NAME} \
       --namespace=kube-system \
       --name=aws-load-balancer-controller \
       --role-name "AmazonEKSLoadBalancerControllerRole" \
       --attach-policy-arn=${POLICY_NAME} \
       --override-existing-serviceaccounts \
       --approve
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-load-balancer-controller --namespace kube-system

NAMESPACE      NAME                            ROLE ARN
kube-system    aws-load-balancer-controller    arn:aws-cn:iam::111122223333:role/AmazonEKSLoadBalancerControllerRole
```

```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get serviceaccounts --all-namespaces  -o wide | grep aws-load-balancer-controller

kube-system       aws-load-balancer-controller        1         94s
```

## 7.Install certManager
安装配置certManager，并检查
```
kubectl apply --validate=false -f cert-manager.yaml
```
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-689bbbbd69-jqvph              1/1     Running   0          10s
cert-manager-cainjector-6d89f558bb-hw6q7   1/1     Running   0          10s
cert-manager-webhook-6f46d85b5f-wq7p2      1/1     Running   0          10s
```

## 8.CreateAWS Load Balance Controller
修改配置文件中的Cluster Name等,先查看，后修改
```
more v2_4_1_full.yaml  | grep -i Cluster-Name -A 5 -B 3;

eval sed -i 's/your-cluster-name/${CLUSTER_NAME}/g' v2_4_1_full.yaml;
sed -i '/- --ingress-class=alb/ i\        - --enable-shield=false\n        - --enable-waf=false\n        - --enable-wafv2=false' v2_4_1_full.yaml
```

检查修改好的yaml
```
[ec2-user@ip-172-31-1-111 ekslab]$ more v2_4_1_full.yaml  | grep -i Cluster-Name -A 5 -B 3;

spec:
      containers:
      - args:
        - --cluster-name=eksgo04
        - --enable-shield=false
        - --enable-waf=false
        - --enable-wafv2=false
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.4.1
```
使用修改好的yaml文件部署AWS Load Balance Controller
```
kubectl apply -f v2_4_1_full.yaml
```
在执行上面命令的过程中可能会碰到如下报错
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl apply -f v2_4_1_full.yaml
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
......
ingressclass.networking.k8s.io/alb created
error: unable to recognize "v2_4_1_full.yaml": no matches for kind "IngressClassParams" in version "elbv2.k8s.aws/v1beta1"
```
执行如下命令后，再重新执行上面的命令
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created


[ec2-user@ip-172-31-1-180 eks_1_22]$ kubectl apply -f v2_4_1_full.yaml
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws configured
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws configured
......
issuer.cert-manager.io/aws-load-balancer-selfsigned-issuer unchanged
mutatingwebhookconfiguration.admissionregistration.k8s.io/aws-load-balancer-webhook configured
validatingwebhookconfiguration.admissionregistration.k8s.io/aws-load-balancer-webhook configured
ingressclassparams.elbv2.k8s.aws/alb created
ingressclass.networking.k8s.io/alb unchanged
```

检查确认AWS Load Balance Controller是否工作
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o aws-load-balancer-controller[a-zA-Z0-9-]+) | grep -i success
I0319 11:57:51.080612       1 leaderelection.go:252] successfully acquired lease kube-system/aws-load-balancer-controller-leader

[ec2-user@iip-172-31-1-111 ekslab]$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           7m15s
```

## 9.Create test application
```
curl -OL https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/examples/2048/2048_full.yaml

kubectl apply -f 2048_full.yaml
```

## 10.Check Ingress
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl get ingress -A
NAMESPACE   NAME           CLASS    HOSTS   ADDRESS                                                                           PORTS   AGE
game-2048   ingress-2048   <none>   *       k8s-game2048-ingress2-20ccadf1ec-1671828624.cn-northwest-1.elb.amazonaws.com.cn   80      46s
```

## 11.Troubleshooting, check logs if there is an issue
```
kubectl logs -n kube-system deployment.apps/aws-load-balancer-controller
```
If the service account doesn’t get IAM role, you can
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl annotate serviceaccount aws-load-balancer-controller -n kube-system eks.amazonaws.com/role-arn=arn:aws-cn:iam::<your_account_id>:role/eksctl-eksgo04-addon-iamserviceaccount-kube-Role1-1JN8QIS4R9C3
serviceaccount/aws-load-balancer-controller annotated
```
```
[ec2-user@ip-172-31-1-111 ekslab]$ kubectl describe serviceaccounts aws-load-balancer-controller -n kube-system
Name:                aws-load-balancer-controller
Namespace:           kube-system
Labels:              app.kubernetes.io/component=controller
                     app.kubernetes.io/name=aws-load-balancer-controller
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::<your_account_id>:role/eksctl-eksgo04-addon-iamserviceaccount-kube-Role1-1JN8QIS4R9C3

Image pull secrets:  <none>
Mountable secrets:   aws-load-balancer-controller-token-gxfqg
Tokens:              aws-load-balancer-controller-token-gxfqg
Events:              <none>
```

Reference:  
[https://docs.amazonaws.cn/eks/latest/userguide/aws-load-balancer-controller.html](https://docs.amazonaws.cn/eks/latest/userguide/aws-load-balancer-controller.html)   
[https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)   
[https://cert-manager.io/docs/installation/](https://cert-manager.io/docs/installation/)
