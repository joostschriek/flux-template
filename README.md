# flux-template

## **Prerequisites**
### **Install kubectl client**

```
Download Instructions: https://kubernetes.io/docs/tasks/tools/install-kubectl/
```

### **Install helm3 client**

```
Download Instructions: https://github.com/helm/helm/releases
```

### **Install kubeseal client**

```
Download Instructions: https://github.com/bitnami-labs/sealed-secrets/releases
```

### **Update the template address**

Note that fluxcd works better with a ssh url.
```
$ sed -i.bak 's/git@github.com:ahanafy\/flux-template/<your git repo>/' flux-patch.yaml
```

## **Deploy**

When running in GKE, we have to setup a clusterrolebinding. This can be done by running
```
./scripts/flux-gke-prep.sh
```

You can run the init script without specifying the repo (it will look at the remote origin used). 
```
./scripts/flux-init.sh
```

If you checked out over https, flux really likes ssh, so you should run the init script with the repository argument.
```
./scripts/flux-init.sh -r git@github.com:username/reponame.git
```

## **Bookinfo**

To check out the bookinfo test page and run sustained GET requests

```
INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://$INGRESS_HOST/productpage"
watch -d "curl -I --request GET --url http://${INGRESS_HOST}/productpage 2>/dev/null | head -n 1"
```

To view the above "bookinfo" requests in Kiali:
```
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```
Then browse to http://localhost:20001

## **Useful commands**
```
kubectl -n fluxcd logs -f deployment/fluxcd --since=2m
kubectl logs -f deploy/helm-operator -n fluxcd --since=2m
kubectl logs -f deploy/helm-operator -n fluxcd --since=10m | grep 'resource conflict'
kubectl port-forward svc/kiali 20001:20001 -n istio-system
kubectl get events --sort-by='{.lastTimestamp}'
kubectl describe svc istio-ingressgateway -n istio-system
kubectl get svc istio-ingressgateway -n istio-system
```

## **Backup SealedSecrets Public/Private keypair**
```
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >master.key
```

## **To backup from Disaster**

Using the master.key file from the above backup step, replace the newly created secrets (from a freshly installed SealedSecrets deployment) and restart the controller

```
$ kubectl apply -f master.key
$ kubectl delete pod -n kube-system -l name=sealed-secrets-controller
```

## **Get Sealed Secret Cert**
```
kubeseal --fetch-cert \
--controller-name=sealed-secrets \
--controller-namespace=kube-system \
> pub-cert.pem
```

## **Create Sealed Secrets**
```
$ kubectl create secret generic test-secret -n mynamespace \
--from-literal=username='my-app' --from-literal=password='39528$vdg7Jb' \
--dry-run -oyaml | \
kubeseal --cert pub-cert.pem --format yaml \
> mysealedsecret.yaml
```

### **Create Sealed Secret with specific scope**
_In this example i use jq to add annotation to the initial manifest_

https://github.com/bitnami-labs/sealed-secrets#scopes

```
$ kubectl create secret generic test-secret -n default \
--from-literal=username='my-app' --from-literal=password='39528$vdg7Jb' \
--dry-run -ojson | \
jq '.metadata += {"annotations":{"sealedsecrets.bitnami.com/cluster-wide":"true"}}' | \
kubeseal --cert pub-cert.pem --format yaml \
> mysealedsecret.yaml
```

## To reconnect fluxcd to git repo

### If you need to print the public key again to load into GitHub
```
kubectl get secrets/fluxcd-git-deploy -n fluxcd -o jsonpath='{.data.identity}'  | base64 -d | ssh-keygen -f /dev/stdin -y
```