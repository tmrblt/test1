## Chain of Trust

**1. create directory structure** \
sub-ca acting as intermediate certificate authority authorized by parent root
- root-ca
- sub-ca (intermediate authority)
- server key pairs

create directories and set permission
```
mkdir -p ca/{root-ca,sub-ca,server}/{private,certs,newcerts,crl,csr}
chmod -v 700 ca/{root-ca,sub-ca,server}/private
tree ca
```

**2. Private Keys** 

**root-ca:** Generate private key for root-ca with aes256 encryption and with key size 4096
```
openssl genrsa -aes256 -out ca/root-ca/private/ca.key 4096
```
**sub-ca:** Generate private key for sub-ca with aes256 encryption and with key size 4096
```
openssl genrsa -aes256 -out ca/sub-ca/private/sub-ca.key 4096
```
**servers:** create servers private key without passphrase and with key size 2048
```
openssl genrsa -out ca/server/private/server.key 2048
```

**3. Create Root CA Certificate** 
- Public Keys are generated from Private Keys
- check  the contents of root-ca.conf
- create sub-ca.conf from root-ca.conf, change dir value

Create root-ca.conf, edit default directory, policy to strict and country, state etc
```
vi ca/root-ca/root-ca.conf
```
Create root-ca certficate
```
openssl req -config ca/root-ca/root-ca.conf -key ca/root-ca/private/ca.key \
-new -x509 -days 7500 -sha256 -extensions v3_ca -out ca/root-ca/certs/ca.crt
```
```
Enter pass phrase for ca/root-ca/private/ca.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [TR]:
State or Province Name [Ankara]:
Locality Name []:
Organization Name [Home Ltd]:
Organizational Unit Name []:
Common Name []:root-ca.home.local
Email Address []:
```

View root CA certificate and verify issuer and subjects are same
```
openssl x509 -noout -in ca/root-ca/certs/ca.crt -text
```

Create sub-ca.conf for intermediate authority
```
cp ca/root-ca/root-ca.conf ca/sub-ca/sub-ca.conf
```

Edit sub-ca.conf
```
vi ca/sub-ca/sub-ca.conf
- dir = /root/ca/sub-ca
- policy = policy_loose
- private_key = $dir/private/sub-ca.key
- certificate = $dir/certs/sub-ca.crt
```
Create Certificate Sign Request\
`sha256:` message digest
```
openssl req -config ca/sub-ca/sub-ca.conf -new \
-key ca/sub-ca/private/sub-ca.key -sha256 -out ca/sub-ca/csr/sub-ca.csr
```

Create Certificate with passphrase
```
openssl ca -config ca/root-ca/root-ca.conf -extensions v3_intermediate_ca -days 3650 \
-notext -in ca/sub-ca/csr/sub-ca.csr -out ca/sub-ca/certs/sub-ca.crt
```

- check new cert and verify that issued by root-ca and subject is sub-ca 
- Now we get key-pair for intermediate/subordinate CA
```
openssl x509 -noout -text -in ca/sub-ca/certs/sub-ca.crt
```

**4. Creating Server Certificates** 

create csr with '*common name: server.home.local*'
```
openssl req -key ca/server/private/server.key -new -sha256 -out ca/server/csr/server.csr
```

create certificate 
```
openssl ca -config ca/sub-ca/sub-ca.conf -extensions server_cert -days 365 \
-notext -in ca/server/csr/server.csr -out ca/server/certs/server.crt
```

create chained server certificate with sign authority
```
cat ca/server/certs/server.crt ca/sub-ca/certs/sub-ca.crt > chained.crt
```

**5. Test PKI Structure**
```
echo "127.0.0.2 server.home.local" >> /etc/hosts
ping server.home.local
```

Start a server instance
```
openssl s_server -accept 443 -www -key ca/server/private/server.key \
-cert ca/server/certs/server.crt -CAfile ca/sub-ca/certs/sub-ca.crt
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
sudo cp ca/root-ca/certs/ca.crt /etc/pki/ca-trust/source/anchors/ca-test.crt
sudo update-ca-trust extract
```

access the service
```
curl https://server.home.local
```