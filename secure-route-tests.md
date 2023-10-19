## Secure Route and Master API Certificate Tests 

**Create certificates**
1. create directory for certificates
```
mkdir certs
```
2. generate private-key for CA
```
openssl genrsa -des3 -out certs/myCA-key.pem 2048
```
3. generate root certificate for CA
```
openssl req -x509 -new -nodes -key certs/myCA-key.pem -sha256 -days 1825 \
-out certs/myCA-cert.pem
```
4. copy root-ca certificate to trust
```
sudo cp certs/myCA-cert.pem /etc/pki/ca-trust/source/anchors/myCA-cert.pem
sudo update-ca-trust extract
```
5. generate private key for wildcard-api
```
openssl genrsa -out certs/wildcard-api-key.pem 2048
```
6. generate CSR for wildcard-api
```
openssl req -new -key certs/wildcard-api-key.pem -out certs/wildcard-api.csr
```
7. create x509v3 extension config for api and *.apps
```
cat << EOF > certs/wildcard-api.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.apps-crc.testing
DNS.2 = api.crc.testing

EOF
```
8. create certificate using CSR, CA private-key and CA cert
```
openssl x509 -req -in certs/wildcard-api.csr -CA certs/myCA-cert.pem \
-CAkey certs/myCA-key.pem -CAcreateserial -out certs/wildcard-api-cert.pem \
-days 365 -sha256 -extfile certs/wildcard-api.ext
```
9. view wildcardapi cert and check DNS
```
openssl x509 -in certs/wildcard-api-cert.pem -text -noout
```
10. combine certificates
```
cat certs/wildcard-api-cert.pem certs/myCA-cert.pem > certs/combined-cert.pem
```

**add CA certificate to OCP trust store**
1. create config map for certificate in openshift-config namespace
```
oc create configmap combined-certs --from-file ca-bundle.crt=certs/combined-cert.pem -n openshift-config
```
2. configure cluster proxy to use the new configuration map
```
oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"combined-certs"}}}'
```

**change ingress certificate**
1. create a new tls-secret in openshift-ingress namespace with certificate and key
```
oc create secret tls custom-tls-bundle --cert certs/combined-cert.pem --key certs/wildcard-api-key.pem -n openshift-ingress
```
2. modify the ingress-controller in openshift-ingress-operator namespace 
```
oc patch ingresscontroller.operator/default -n openshift-ingress-operator --type=merge --patch='{"spec": {"defaultCertificate": {"name": "custom-tls-bundle"}}}'
```
3. verify new router pods in openshift-ingress namespace to confirm the change is successful
```
watch oc get pods -n openshift-ingress
```
4. access the console from web-browser and verify trusted
```
oc whoami --show-console
```
5. create an application with edge/passthrough route and verify cert is trusted
```
oc new-app https://github.com/sclorg/cakephp-ex
```
```
oc create route edge ...
```
**replace master api certificate**
1. create TLS secret for certificate
```
oc create secret tls custom-tls --cert certs/combined-cert.pem --key certs/wildcard-api-key.pem -n openshift-config
```
2. configure API server to use TLS secret
```yaml
cat << EOF > apiserver-patch.yaml
apiVersion: config.openshift.io/v1
kind: APIServer
metadata:
  name: cluster
spec:
  servingCerts:
    namedCertificates:
    - names:
      - api.crc.testing
      servingCertificate:
        name: custom-tls

EOF
```
```
oc apply -f apiserver-patch.yaml
```
3. wait for redeployment finished
```
watch "oc get clusteroperator/kube-apiserver ; \
oc get pods -l app=openshift-kube-apiserver -n openshift-kube-apiserver"
```
```
Unable to connect to the server: x509: certificate signed by unknown authority
oc logout
oc login -u kubeadmin -p 4E2EA-2BJnE-zbTep-zwcsk https://api.crc.testing:6443
```
