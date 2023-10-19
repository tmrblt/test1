## Simple CA with SAN

**0. Create directory**
create dirs
```
mkdir certs
```

**1. CA**\
Generate private-key for CA
```
openssl genrsa -des3 -out certs/myCA.key 2048
```

generate root certificate for CA
```
openssl req -x509 -new -nodes -key certs/myCA.key -sha256 -days 1825 \
-out certs/myCA.pem
```

**2. Server**\
generate private key for server
```
openssl genrsa -out certs/server.key 2048
```

generate CSR for server
```
openssl req -new -key certs/server.key -out certs/server.csr
```

create x509v3 extension config
```
echo < EOF >> certs/server.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = server.home.local

EOF

```

create certificate using CSR, CA private-key and CA cert
```
openssl x509 -req -in certs/server.csr -CA certs/myCA.pem -CAkey certs/myCA.key \
-CAcreateserial -out certs/server.crt -days 365 -sha256 -extfile certs/server.ext
```

**3. Test server certificate**
```
echo "127.0.0.2 server.home.local" >> /etc/hosts
ping server.home.local
```

Start a server instance
```
openssl s_server -accept 443 -www -key certs/server.key \
-cert certs/server.crt -CAfile certs/myCA.pem
```

Check service on another terminal
```
ss -ntl
```

Try to access 
```
curl https://server.home.local
```

copy root-ca certificate to trust
```
sudo cp certs/myCA.pem /etc/pki/ca-trust/source/anchors/myCA-test.pem
sudo update-ca-trust extract
```

access the service
```
curl https://server.home.local
```