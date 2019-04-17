


1. change **--cluster-name=** in alb-ingress-controller.yaml

2. change **--txt-owner-id=** and **--domain-filter** in external-dns.yaml

3. change public subnet in ingress.yaml

4. change host in ingress.yaml



```
kubectl -f apply rbac-role.yaml
kubectl -f apply alb-ingress-controller.yaml

kubectl -f apply deployment.yaml
kubectl -f apply service.yaml
kubectl -f apply ingress.yaml

kubectl -f apply external-dns.yaml
```
