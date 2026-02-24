### kubectl apply commands (in order)

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f mongo-configmap.yaml
kubectl apply -f mongo-express.yaml
```

### kubectl get commands

```bash
kubectl get pod
kubectl get pod --watch
kubectl get pod -o wide
kubectl get service
kubectl get secret
kubectl get all | grep mongodb
```

### kubectl debugging commands

```bash
kubectl describe pod mongodb-deployment-xxxxxx
kubectl describe service mongodb-service
kubectl logs mongo-express-xxxxxx
```

### Give a URL to external service in minikube

```bash
minikube service mongo-express-service
```
