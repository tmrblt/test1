1. Create app
```
oc create -f https://raw.githubusercontent.com/openshift/origin/master/examples/hello-openshift/hello-pod.json
```
2. Create service
```
oc expose pod/hello-openshift
```
3. Create route insecure
```
oc expose svc hello-openshift
```
4. Create ingress - insecure
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: hello-openshift-ingress
 annotations:
   haproxy.router.openshift.io/rewrite-target: / 
spec:
 rules:
   - host: hello-openshift-ingress.apps.ocx.sandbox2677.opentlc.com
     http:
       paths:
         - path: /
           pathType: Prefix 
           backend:
             service:
                name: hello-openshift
                port:
                  number: 8080
```
5. Create ingress - secure
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: secure-hello-openshift-ingress
 annotations:
   haproxy.router.openshift.io/rewrite-target: / 
spec:
  tls:
  - hosts:
    - secure-hello-openshift-ingress.apps.ocx.sandbox2677.opentlc.com
    secretName: custom-tls-bundle
  rules:
   - host: secure-hello-openshift-ingress.apps.ocx.sandbox2677.opentlc.com
     http:
       paths:
         - path: /
           pathType: Prefix 
           backend:
             service:
                name: hello-openshift
                port:
                  number: 8080
```
