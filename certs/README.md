## Setup Certs

The following instructions document, how certificates were generated for this example.

Generate the CA private key:

```bash
openssl genpkey -pass pass:changeme -algorithm RSA -out ca.key -aes256 
```

Generate the CA certificate:

```bash
openssl req -passin pass:changeme -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=US/ST=State/L=City/O=Organization/OU=OrgUnit/CN=RootCA"
```

Generate two certificates, one valid, one expired sharing the same key and subject

```bash
openssl genpkey -pass pass:changeme -algorithm RSA -out cert.key -aes256

openssl req -passin pass:changeme -new -key cert.key -out cert.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=OrgUnit/CN=Intermediate"

# generate a cert valid from 2024 expiring 2030
openssl x509 -passin pass:changeme -req -in cert.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out valid.crt -not_before 20241001000000Z -not_after 20301002000000Z -sha256
# generate a cert valid from 20201 expiring 2022
openssl x509 -passin pass:changeme -req -in cert.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out expired.crt -not_before 20211001000000Z -not_after 20221001000000Z -sha256
```

Generate certificate from valid cert.

```bash
openssl genpkey -pass pass:changeme -algorithm RSA -out server.key -aes256
openssl req -passin pass:changeme -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=OrgUnit/CN=localhost"

openssl x509 -passin pass:changeme -req -in server.csr -CA valid.crt -CAkey cert.key -CAcreateserial -out server.crt -not_before 20241001000000Z -not_after 20301002000000Z -sha256
```

Diff the certs

```bash
cat server.crt | openssl x509 -text > server.txt
cat valid.crt | openssl x509 -text > valid.txt
cat expired.crt | openssl x509 -text > expired.txt
diff valid.txt expired.txt
```


