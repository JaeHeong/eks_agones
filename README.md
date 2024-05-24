# eks_agones

## Terraform apply
- export TF_VAR_region=$AWS_REGION
- export TF_VAR_accountID=''
- export TF_VAR_user_name=''
- terraform init
- terraform apply -target="module.vpc" -auto-approve
- terraform apply -target="module.eks" -auto-approve
- terraform apply --auto-approve

## Edit agones-controller
- kubectl -n agones-system edit deploy agones-controller
- ephemeral-storage: 10010Mi -> 5000Mi

## Kube-Ops-View
- kubectl apply -f kube-ops-viewer/
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
- terraform destroy -target="module.eks_blueprints_addons" --auto-approve
- terraform destroy -target="module.eks" --auto-approve
- terraform destroy --auto-approve

# Reference
- https://community.aws/content/2dhlDEUfwElQ9mhtOP6D8YJbULA/run-kubernetes-clusters-for-less-with-amazon-ec2-spot-and-karpenter#step-6-optional-simulate-spot-interruption
