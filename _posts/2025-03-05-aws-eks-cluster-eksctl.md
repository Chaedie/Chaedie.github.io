---
title: AWS EKS Cluster 설정 - eksctl (+ trouble shooting)
date: 2025-03-05 # HH:MM:SS +/-TTTT
categories: [Devops]
tags: [aws, eks]     # TAG names should always be lowercase

description: AWS EKS Cluster 설정 - eksctl (+ trouble shooting)
toc: true
comments: true
---

### 1. 서론 

두번째 DevOps 프로젝트를 진행하며 AWS EKS Cluster 를 설정해보았다.

첫번째 프로젝트에선 Terraform을 이용한 AWS Network 설정부터 익숙하지 않은 상황에서 EKS 설정, ArgoCD 첫 경험 등 수많은 IAM, RBAC 등으로 이해보단 복잡한 Hands On을 직접 해본다는 데 의미를 두고 진행했었다. 

이번 두번째 프로젝트에선 Network setting 과 Terraform 세팅 자체를 간소화했고, Sonar Qube 또한 별도 설정없이 docker 로 간단하게 띄웠다. 그랬더니 jenkins pipeline, eks, argoCD 에 대해 조금 더 경험해 보는 계기가 되었다.

### 2. EKS 설정 - eksctl

- eks생성, Load Balancer 설정

```bash
eksctl create cluster \
	--name devops-project-2-k8s-eks-cluster \
    --region ap-northeast-2 \
    --node-type t2.medium \
    --nodes-min 2 \
    --nodes-max 2
    
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/docs/install/iam_policy.json

aws iam create-policy \
	--policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

eksctl utils associate-iam-oidc-provider \
	--region=ap-northeast-2 \
    --cluster=devops-project-2-k8s-eks-cluster \
    --approve

eksctl create iamserviceaccount \
	--cluster=devops-project-2-k8s-eks-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve \
    --region=ap-northeast-2

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
	-n kube-system \
    --set clusterName=devops-project-2-k8s-eks-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 3. Prometheus, Grafana 설치

- 설치

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

kubectl create ns monitoring
helm install stable prometheus-community/kube-prometheus-stack -n monitoring

kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

- expose

```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n monitoring
# ClusterIP -> LoadBalancer
kubectl get svc -n monitoring

kubectl edit svc stable-grafana -n monitoring
# ClusterIP -> LoadBalancer
kubectl get svc -n monitoring
```

### 4. ArgoCD 설치

- 설치

```bash
kubectl create ns three-tier
kubectl create secret generic ecr-registry-secret \
  --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson -n three-tier
  
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.3/manifests/install.yaml

# Expose
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Retrieve initial Admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

### 5. Trouble Shooting - EKS 노드 갯수 변경

- 비용절감을 위해 2개인 노드를 1개로 줄여주었다. EKS 의 경우 master node 는 관리형으로 주어지기 때문에 비용절감이 불가능하지만 worker node의 경우 ec2 Instance 로 제공되므로 노드갯수와 타입에 따라 비용을 절감할 수 있다.

eksctl get nodegroup --cluster devops-project-2-k8s-eks-cluster

eksctl scale nodegroup \
	--cluster devops-project-2-k8s-eks-cluster \
    --name ng-7d39f248 \
    --nodes 1 \
    --nodes-min 1 \
    --nodes-max 1


### 6. Trouble Shooting - Dependency Check 도중 Jenkins Server 응답 없음 현상

e2 console 에서 status check 는 2/2 로 정상이지만 request 에는 응답하지 않는 현상이 발생했다. (공식문서 참고)

[Status checks for Amazon EC2 instances - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html)

Monitoring Dashboard 를 보면 CPU Utilization, Storage 스파크가 확인되었다. IO 작업에 대한 부하 때문에 응답이 없는걸로 예상되었다. 마냥 기다려야 되는건지 조치를 취해야하는지 알 수 없어 AWS 에서 제공하는 Youtube 영상을 통해 Trouble Shoot 의 Common Step 을 배워보기로 했다. 

AWS 공식 유튜브 영상 - [How do I troubleshoot an unresponsive website hosted on my EC2 Instance?](https://www.youtube.com/watch?v=xvtoVxk8kWA)

- Health Check 확인
- System Log 확인 for check the kernel panic
- CPU Utilization 확인
- Network in/out 확인
- Storage 확인
- Security Group 확인 (휴먼에러)
- Networking
- VPC
- network ACL
- inbound/outbound rule 확인
- route table 확인
- EIP 확인
- SSH 되는 경우
  - 서비스 확인 `sudo systemctl status httpd`
  - firewall 확인 `sudo firewall-cmd --state`
  - ubuntu 의 경우 `sudo ufw status verbose`
    - allow `sudo ufw allow in 80/tcp`, `sudo ufw allow 443/fcp`
  - /var/log/httpd 경로의 error_log, access_log 확인

httpd 서비스에서의 트러블 슈팅 과정을 보여주는 영상이었는데, 해당 영상을 시청하고 나서 ec2가 정상으로 돌아왔다. reboot 명령 때문인지 단지 작업이 끝이 나서인지는 모르곘지만 우선 Trouble shooting 시의 common step 을 한번 훑어보는 기회가 되었다.

### 7. Trouble Shooting - ArgoCD의 일부 pod 들이 Pending 상태인 상황

- ArgoCD 의 일부 pod들이 pending 상태로 노드에 스케쥴링이 되지 않는 현상 발생
- `kubectl top node` 를 통해 resource 를 확인해보니 memory 58% 가량 사용중임을 확인
- 1개 였던 node 를 2개로 늘려주고 resource 부족으로 스케쥴링 대기 상태인 Pending 상태인 pode들의 경우 delete 해줌으로써 새로운 node 에 배치되도록 했다.
- (노드를 scale out 해주어도 pending state 인 pod 의 경우는 다른 노드로 할당되지 않는다.)
