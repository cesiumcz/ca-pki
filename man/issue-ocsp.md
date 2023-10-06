## Issuing OCSP certificate
```
umask 077
git pull
cd pki/<CA_DIRECTORY>
mkdir -p csr private newcerts
openssl ecparam -genkey -noout -name prime256v1 -out private/ocsp.key.pem && chmod 400 private/ocsp.key.pem
```
Generate request. Set CN to ocsp.example.org, leave email blank
```
openssl req -config ./openssl.cnf -new \
	-key private/ocsp.key.pem \
	-out csr/ocsp.csr.pem
```

Sign the request
```
openssl ca -config ./openssl.cnf \
	-extensions v3_ocsp -days 365 -notext \
	-in csr/ocsp.csr.pem -out certs/ocsp.cert.pem
```

Verify the certificate
```
openssl x509 -noout -text -nameopt utf8 -in certs/ocsp.cert.pem
openssl verify -CAfile certs/ca.fchain-cert.pem certs/ocsp.cert.pem
```
For issuing CA, use fullchain certificate
```
openssl verify -CAfile certs/ca.fchain-cert.pem certs/ocsp.cert.pem
```

[Optional] Git commit
```
git add .
git commit -m "Issued new OCSP certificate."
git push
```
