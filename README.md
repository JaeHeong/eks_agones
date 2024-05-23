# eks_agones

## Terraform apply
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

## Terraform destroy
- terraform destroy -target="module.eks_blueprints_addons" --auto-approve
- terraform destroy -target="module.eks" --auto-approve
- terraform destroy --auto-approve
