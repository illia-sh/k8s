##### Jenkins + nginx ingress with automated TLS with:
https://letsencrypt.org/
https://github.com/jetstack/cert-manager

#### Jenkins
##### create jenkins namespace/service/deployment
```
kubectl apply -f jenkins/jenkins-ns.yml
kubectl apply -f jenkins/services/front-jenkins-svc.yaml
kubectl apply -f jenkins/deployments/jenkins-dep.yml
```

##### get root password for jenkins from pod logs
```
kubectl get pods -n jenkins -w # check pod name
kubectl logs jenkins-597fcbbd98-sl8ck -n jenkins | grep password -A3
```
#### Nginx
###### create nginx-ingress namespace
```
kubectl apply -f nginx/namespace.yml
```
##### create default http backend:
```
kubectl apply -f nginx/default-deployment.yaml
kubectl apply -f nginx/default-service.yaml
```
##### create nginx ingress controller deployment:
```
kubectl apply -f nginx/configmap.yaml
kubectl apply -f nginx/service.yaml
kubectl apply -f nginx/deployment.yaml
```
##### check external ip
```
kubectl describe svc nginx --namespace nginx-ingress
```
#### enable TLS
##### enable cert-manager
```
kubectl create -f cert-manager`
```
To trigger TLS to be enabled, and for cert-manager to obtain a Let’s Encrypt certificate for it, we need to update the Ingress resource.

    A tls stanza is required with a host field to match in the SNI header, and a secretName where the certificates will be stored by cert-manager and fetched by the ingress controller
    The presence of the certmanager.k8s.io/cluster-issuer annotation tells cert-manager to automatically provision a certificate for this Ingresss resource. Its value specifies the CA to ask to sign the certificate.

##### Edit in file or manually via k8s way
```
kubectl edit ing jenkins-ingress -n jenkins
```
 Add your hostname e.g. mycluster.example.com and the certmanager.k8s.io/cluster-issuer: "letsencrypt-prod" annotation.
##### Important: cert-manager will only provision certificates for Ingress resources with this annotation.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - shpak.co.uk
    secretName: jenkins-tls
  rules:
  - host: shpak.co.uk # valid dns name
    http:
      paths:
      - path: /
        backend:
          serviceName: jenkins
          servicePort: 443
```

Now we can watch the cert-manager logs and see the certificate request taking place (via the ACME protocol.
```
$ kubectl logs cert-manager-pod --namespace=cert-manager -c ingress-shim
$ kubectl logs cert-manager-pod --namespace=cert-manager -c cert-manager
```
Once cert-manager has finished its magic, you should see two certificates in a Secret. NGINX knows where to find these certificates and will be ready-configured to serve traffic with TLS on the domain.
```
$ kubectl get secret --namespace=jenkins
```
https://yourhostname.com
