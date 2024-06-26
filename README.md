# eks_agones

## Terraform apply
- export TF_VAR_region=$AWS_REGION
- export TF_VAR_accountID=''
- export TF_VAR_user_name=''
- terraform init
- terraform apply -target="module.vpc" -auto-approve && terraform apply -target="module.eks" -auto-approve && terraform apply --auto-approve
- aws eks --region $AWS_REGION update-kubeconfig --name spot-and-karpenter

## Kubectl CheatSheet
- source <(kubectl completion bash) # bash-completion 패키지를 먼저 설치한 후, bash의 자동 완성을 현재 셸에 설정한다
- echo "source <(kubectl completion bash)" >> ~/.bashrc # 자동 완성을 bash 셸에 영구적으로 추가한다
- alias k=kubectl
- complete -o default -F __start_kubectl k

## Edit agones-controller for test
- ephemeral-storage 10100Mi -> 5000Mi
- kubectl patch deployment agones-controller --namespace agones-system --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/ephemeral-storage", "value": "5000Mi"}, {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/ephemeral-storage", "value": "5000Mi"}]'

## Kube-Ops-View
- kubectl apply -f kube-ops-viewer/
- kubectl patch service kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'
- Wait a minute to deploy the LoadBalancer
- kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'

## Create pod using Karpenter 
- export CLUSTER_NAME=$(terraform output -raw cluster_name)
- export KARPENTER_NODE_IAM_ROLE_NAME=$(terraform output -raw node_instance_profile_name)

```
# Create NodePool
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default 
spec:  
  template:
    metadata:
      labels:
        intent: apps
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: "karpenter.k8s.aws/instance-cpu"
          operator: Gt
          values: ["4"]
        - key: "karpenter.k8s.aws/instance-memory"
          operator: Gt
          values: ["8191"] # 8 * 1024 - 1
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      kubelet:
        systemReserved:
          cpu: 100m
          memory: 100Mi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 168h # 7 * 24h = 168h
  
---   

apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  subnetSelectorTerms:          
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  role: ${KARPENTER_NODE_IAM_ROLE_NAME}
  tags:
    project: build-on-aws
    IntentLabel: apps
    KarpenterNodePoolName: default
    NodeType: default
    intent: apps
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF
```
```
# Need below nodeSelector when deploy pod
spec:
  nodeSelector:
    intent: apps
    karpenter.sh/capacity-type: spot
```

## Show Logging Karpenter
- alias kl='kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter --all-containers=true -f --tail=20';
- kl

## Terraform destroy
- terraform destroy -target="module.eks_blueprints_addons" --auto-approve && terraform destroy -target="module.eks" --auto-approve && terraform destroy --auto-approve

## eks-node-viewer
- eks-node-viewer

# Reference
- https://community.aws/content/2dhlDEUfwElQ9mhtOP6D8YJbULA/run-kubernetes-clusters-for-less-with-amazon-ec2-spot-and-karpenter#step-6-optional-simulate-spot-interruption
