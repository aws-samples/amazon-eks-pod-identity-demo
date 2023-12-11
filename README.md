# Amazon EKS Pod Identity Demo

In this walkthrough, we will demonstrate how to use the new Amazon EKS Pod Identity feature to securely provide AWS access to kubernetes pods.

## Solution Architecture

![Solution Architecture](assets/eks-pod-identity-arch.jpg)

## Pre-requisites

* An AWS Account
* export AWS_REGION=us-west-2 # Replace with your AWS region
* export CLUSTER_NAME=eks-pod-identity-demo # Replace with your cluster name
* export AWS_ACCOUNT=123456789012 # Your AWS Account number
* [AWS CLI](https://aws.amazon.com/cli/) and credentails from both AWS Accounts, alternatively use [AWS CloudShell](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html#how-to-get-started)
* [eksctl](https://eksctl.io/) - a simple CLI tool for creating and managing Amazon EKS clusters
* [git](https://github.com/git-guides/install-git)
  ```shell
    git clone https://github.com/aws-samples/amazon-eks-pod-identity-demo
    cd amazon-eks-pod-identity-demo
  ```

## Setup

Lets start by creating an Amazon EKS Cluster using eksctl with the new eks-pod-identity-agent addon.

```shell
eksctl create cluster -f cluster.yaml
```

Wait for cluster creation to be complete and verify if eks-pod-identity-agent addon is running on the cluster and the worker nodes.

```shell
aws eks describe-addon --cluster-name ${CLUSTER_NAME} --region ${AWS_REGION} --addon-name eks-pod-identity-agent
```
```json
{
    "addon": {
        "addonName": "eks-pod-identity-agent",
        "clusterName": "eks-np-demo",
        "status": "ACTIVE",
        "addonVersion": "v1.0.0-eksbuild.1",
        "health": {
            "issues": []
        },
        "addonArn": "arn:aws:eks:us-west-2:123456789012:addon/eks-np-demo/eks-pod-identity-agent/6cc61b38-f8b4-a9b3-dc86-82f9828c6ca9",
        "createdAt": "2023-12-04T15:08:06.746000-05:00",
        "modifiedAt": "2023-12-04T15:08:39.106000-05:00",
        "tags": {}
    }
}
```

```shell
kubectl get pods -n kube-system --selector app.kubernetes.io/name=eks-pod-identity-agent
```
```output
NAME                           READY   STATUS    RESTARTS   AGE
eks-pod-identity-agent-64q42   1/1     Running   0          80m
eks-pod-identity-agent-tkxb4   1/1     Running   0          79m
eks-pod-identity-agent-zq9k2   1/1     Running   0          79m
```

Deploy a sample python flask application that utilizes IAM credentials to access Amazon S3 buckets. We will use EKS Pod Identity feature to associate an IAM role to the kubernetes service account assigned to the deployment. Lets start by creating an IAM role with S3 readonly policy.

```shell
export POD_ROLE_ARN=$(aws iam create-role --role-name s3-app-eks-pod-identity-role \
 --assume-role-policy-document file://eks-pod-identity-trust-policy.json \
 --output text --query 'Role.Arn')

aws iam attach-role-policy --role-name s3-app-eks-pod-identity-role \ 
--policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess 
```

Now, associate the IAM role with the EKS Pod Identity by calling create-pod-identity-association.

```shell
aws eks create-pod-identity-association \
--cluster-name $CLUSTER_NAME \
--namespace s3-app-ns \
--service-account s3-app-sa \
--role-arn $POD_ROLE_ARN \
--region $AWS_REGION
```

![Amazon EKS Pod Identity Console](assets/eks-pod-identity-console.png)

You can use this API to create multiple pod identity associations on an IAM role across namespaces, Amazon EKS clusters with in an AWS Account. 

Lets deploy a sample python flask application that utilizes boto3 sdk to interact with Amazon S3 resources. This is deployed in s3-app-ns namespace and with s3-app-sa service account.

```shell
kubectl apply -f s3-app.yaml
```
```output
namespace/s3-app-ns created
serviceaccount/s3-app-sa created
deployment.apps/s3-app-deployment created
service/s3-app-svc created
```

Wait for the pods and LoadBalancer service to become ready and fetch the LoadBalancer url using the below command:

```shell
kubectl get all -n s3-app-ns
```
```output
NAME                                    READY   STATUS        RESTARTS   AGE
pod/s3-app-deployment-79c7c5c696-nrfsb  1/1     Running       0          29s
pod/s3-app-deployment-79c7c5c696-rt562  1/1     Running       0          22s

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP                                                                      PORT(S)        AGE
service/s3-app-svc   LoadBalancer   172.20.10.54   a3c4874577a65486aa23c5508ee3c3f7-1a804a21e2f25d95.elb.us-west-2.amazonaws.com    80:31118/TCP   5s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/s3-app-deployment   2/2     2            2           42m

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/s3-app-deployment-79c7c5c696   2         2         2       29s
```

```shell
export LB_URL=$(kubectl get svc -n s3-app-ns s3-app-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```
```output
curl http://$LB_URL/list-buckets
[LIST OF BUCKETS]
```

This demonstrates how the EKS workloads can utilize the new EKS Pod Identity feature to securely access other AWS resources using the temporary IAM credentials.


## Clean up

```shell
eksctl delete cluster -f cluster.yaml
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

