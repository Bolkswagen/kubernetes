# Kubernetes ♬
---

## 1. EKS
Base user guide : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/what-is-eks.html  
<br/>

#### ▣ Metrics-server installation
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
```bash
kubectl get deployment metrics-server -n kube-system
```

#### ▣ Metrics-server installation (high-availability)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/high-availability.yaml
```
1.22 버전 기준 다음과 같은 메세지 출력
```bash
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
```
```bash
kubectl get deployment metrics-server -n kube-system
```
[ Reference ]  
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html  
https://github.com/kubernetes-sigs/metrics-server/releases  
<br/>

#### ▣ Cluster autoscaler
1. Create IAM policy
```bash
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
2. (클러스터에 대한 IAM OIDC provider가 없는 경우) [Create IAM OIDC provider](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
3. Create EKS iamserviceaccount
```bash
eksctl create iamserviceaccount \
  --cluster=<my-cluster> \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/<AmazonEKSClusterAutoscalerPolicy> \
  --override-existing-serviceaccounts \
  --approve
```
4. Deploy cluster autoscaler
```bash
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
Yaml 파일을 수정합니다.
```bash
vi cluster-autoscaler-autodiscover.yaml
```
위 과정에서 SA를 생성하였으므로 상단의 SA 매니패스트부분을 주석처리합니다.
```yaml
// cluster-autoscaler-autodiscover.yaml
---
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   labels:
#     k8s-addon: cluster-autoscaler.addons.k8s.io
#     k8s-app: cluster-autoscaler
#   name: cluster-autoscaler
#   namespace: kube-system
---
```
① Deployment manifast 부분에 `cluster-autoscaler.kubernetes.io/safe-to-evict: 'false'` 어노테이션을 추가  
② `<YOUR CLUSTER NAME>` 을 현재 클러스터명으로 변경  
③ 다음 옵션을 추가 `--balance-similar-node-groups` `--skip-nodes-with-system-pods=false`  
```yaml
// cluster-autoscaler-autodiscover.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'false'     # ① 어노테이션추가
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.2
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>   # ② 클러스터명으로 변경
            - --balance-similar-node-groups         # ③ 옵션추가
            - --skip-nodes-with-system-pods=false   # ③ 옵션추가
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt #/etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```

About `cluster-autoscaler.kubernetes.io/safe-to-evict` annotation  
해당 어노테이션이 false로 들어가 있는 파드의 노드는 줄어들지 않습니다. 이는 일반적인 파드에도 적용될 수 있으며 위의 경우는 클러스터 오토스케일러 파드에 적용됩니다.


|제목|내용|설명|
|---|---|---|
|테스트1|테스트2|테스트3|
|테스트1|테스트2|테스트3|
|테스트1|테스트2|테스트3|

수정을 완료하였으면 배포합니다.  
```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```




[ Reference ]  
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/autoscaling.html  
<br/>

















