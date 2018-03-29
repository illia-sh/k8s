# k8s



#### get external ip adresses from nodes:
```
$kubectl get nodes --selector=kubernetes.io/role!=master -o jsonpath={.items[*].status.addresses[?\(@.type==\"ExternalIP\"\)].address}
```
