# kubernetes-aws

### Installation

```
sudo yum update && sudo yum -y upgrade

sudo yum install python-pip

sudo pip install awscli --upgrade --user

sudo yum install git vim unzip


aws configure




cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo mv kubernetes.repo /etc/yum.repos.d/

sudo yum install -y kubectl


```


### Create Cluster
```
aws eks create-cluster \
		--name test \
		--role-arn arn:aws:iam::123456789:role/test-eks \
		--resources-vpc-config subnetIds=subnet-123456,subnet-654321,subnet-111111,securityGroupIds=sg-abcde12345 \
		--region eu-west-1

aws eks update-kubeconfig --name test --region eu-west-1
```

### Create Nodes Using Cloudformation

```sh
aws cloudformation create-stack \
                --stack-name lucas-eks-nodes \
                --template-url https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-30/amazon-eks-nodegroup.yaml \
                --parameters \
                        ParameterKey=ClusterName,ParameterValue=test \
                        ParameterKey=NodeGroupName,ParameterValue=test \
                        ParameterKey=NodeAutoScalingGroupMaxSize,ParameterValue=1 \
                        ParameterKey=NodeAutoScalingGroupMinSize,ParameterValue=0 \
                        ParameterKey=NodeInstanceType,ParameterValue=t2.small \
                        ParameterKey=ClusterControlPlaneSecurityGroup,ParameterValue=sg-1234567 \
                        ParameterKey=KeyName,ParameterValue=lucasko-key \
                        ParameterKey=NodeImageId,ParameterValue=ami-0c7a4976cb6fafd3a \
                        ParameterKey=VpcId,ParameterValue=vpc-12345678\
                        ParameterKey=Subnets,ParameterValue=subnet-123456\\,subnet-654321 \
                --capabilities CAPABILITY_IAM \
                --region eu-west-1
```

>> You can find NodeImageId in [Launching Amazon EKS Worker Nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-workers.html)

### Configure Permission of Nodes

After compeleting the cloudformation.

Obtain the value of role arn.

```
aws cloudformation describe-stacks --stack-name lucas-eks-nodes | grep NodeInstanceRole | awk '{print $7}'

arn:aws:iam::1111111111111:role/lucas-eks-nodes-NodeInstanceRole-5566
```

Download and edit the autherization file.

```sh                
wget https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-30/aws-auth-cm.yaml

vim aws-auth-cm.yaml

```

Replace the rolearn with above value we found.

Content of **aws-auth-cm.yaml**

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::1111111111111:role/lucas-eks-nodes-NodeInstanceRole-5566
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

```

> Change rolearn to the value of NodeInstanceRole
> 
> AWS -> CloudFormation -> Your Stack -> Outputs -> Value

Make it work.

```
$ kubectl apply -f aws-auth-cm.yaml
configmap/aws-auth changed

$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-255-75.eu-west-1.compute.internal   Ready    <none>   51m   v1.10.3

$ kubectl config current-context
arn:aws:eks:eu-west-1:1111111111111:cluster/test
```





### EKS: EBS Volume


```sh
aws ec2 create-volume --region eu-west-1 --availability-zone eu-west-1a --size 10 --volume-type gp2
```


```sh
vim my-pv.yml
```

```sh
kind: PersistentVolume
apiVersion: v1
metadata:
  name: my-pv
  labels:
    type: amazonEBS
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2  
  awsElasticBlockStore:
    volumeID: <your-volume-id>
    fsType: ext4
```



```sh
kubectl create -f my-pv.yml

kubectl get pv
```


```sh
vim my-pvc.yml
```


```sh
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pvc
  labels:
    type: amazonEBS
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

 
```sh
kubectl create -f my-pvc.yml

kubectl get pvc
```


### Deployment vs StatefulSet
 
 * **Deployment** - You specify a PersistentVolumeClaim that is shared by all pod replicas. In other words, shared volume. The backing storage obviously must have ReadWriteMany or ReadOnlyMany accessMode if you have more than one replica pod.

 * **StatefulSet** - You specify a volumeClaimTemplates so that each replica pod gets a unique PersistentVolumeClaim associated with it. In other words, no shared volume. Here, the backing storage can have ReadWriteOnce accessMode.

 
So if your application is stateful or if you want to deploy stateful storage on top of Kubernetes use a **StatefulSet**.

If your application is stateless or if state can be built up from backend-systems during the start then use **Deployments**.

