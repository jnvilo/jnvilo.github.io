### Open shell on container 

```
kubectl exec -i -t my-pod --container main-app -- /bin/bash
```

To know the container name:

```
kubectl get pods POD_NAME_HERE -o jsonpath='{.spec.containers[*].name}'
```

