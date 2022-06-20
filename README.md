# Kubernetes ♬
---

## 1. EKS
Base user guide : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/what-is-eks.html
<br/><br/>

##### ▣ Metrics-server installation
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
```
kubectl get deployment metrics-server -n kube-system
```

##### ▣ Metrics-server installation (high-availability)
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
```
1.22 버전 기준 다음과 같은 메세지 출력
```
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
```
```
kubectl get deployment metrics-server -n kube-system
```
[ Reference ]  
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html  
https://github.com/kubernetes-sigs/metrics-server/releases
<br/><br/>

##### ▣ Cluster autoscaler
1. Create IAM policy
```
aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document \
'{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}'
```






[ Reference ]
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/autoscaling.html


















