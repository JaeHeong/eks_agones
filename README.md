# eks_agones

- kubectl -n agones-system edit deploy agones-controller
- ephemeral-storage: 10010Mi -> 5000Mi
- kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
