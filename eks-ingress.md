## Install NGINX Ingress Controller (Manifests)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```
### Verify installation:
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```
```bash
kubectl apply -f ingress.yaml
```
