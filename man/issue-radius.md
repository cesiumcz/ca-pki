## Issuing RADIUS server certificate
### RADIUS server
On RADIUS server, generate the key and request using `openssl.cnf` from `pki/dvtls/openssl.cnf`
```
umask 077
git pull
cd pki/dvtls
mkdir -p csr private newcerts
openssl ecparam -genkey -name prime256v1 | openssl ec -aes256 -out private/radius.example.lan_<YEAR>.key.pem && chmod 400 private/radius.example.lan_<YEAR>.key.pem
```

Set RADIUS server DNS name
```
sed -i 's#<RADIUS_DNS>#radius.example.lan#g' openssl.cnf
```

Create a request. Set CN to radius.example.lan, leave email blank
```
openssl req -config ./openssl.cnf -new \
	-key private/radius.example.lan_<YEAR>.key.pem \
	-outform DER -out csr/radius.example.lan_<YEAR>.csr.der
```

Transfer `radius.example.lan_<YEAR>.csr.der` to signing machine (`pki/dvtls/csr/`)

### Signing device
```
cd pki/dvtls
```

**Check and verify all distinguished name fields**
```
openssl req -noout -text -nameopt utf8 -inform DER -in csr/radius.example.lan_<YEAR>.csr.der
```

> **_NOTE:_**  x509v3 extensions are intentionally not copied automatically to the certificate (`-copy_extensions`) due to high risk of introducing dangerous values (CA:TRUE / keyUsage).  
Thus, extended attributes (like e-mail addresses, DNS names or IP addresses) must be manually verified and rewritten by man.**  

Set RADIUS server DNS name
```
sed -i 's#<RADIUS_DNS>#radius.example.lan#g' openssl.cnf
```

Sign the request using DV TLS CA private key in HSM
```
openssl ca -config ./openssl.cnf \
      -extensions v3_radius_server -days 730 -notext \
      -engine pkcs11 -keyform engine -keyfile 0:02 \
      -inform DER -in csr/radius.example.lan_<YEAR>.csr.der -out certs/radius.example.lan_<YEAR>.cert.pem
```

[Alternative] Sign the request using DV TLS CA private key in computer
```
openssl ca -config ./openssl.cnf \
	-extensions v3_radius_server -days 730 -notext \
	-inform DER -in csr/radius.example.lan_<YEAR>.csr.der -out certs/radius.example.lan_<YEAR>.cert.pem
```

Verify the certificate
```
openssl x509 -noout -text -nameopt utf8 -in certs/radius.example.lan_<YEAR>.cert.pem
openssl verify -CAfile certs/ca.fchain-cert.pem certs/radius.example.lan_<YEAR>.cert.pem
```

Create fullchain certificate
```
cat certs/radius.example.lan_<YEAR>.cert.pem certs/ca.fchain-cert.pem > certs/radius.example.lan_<YEAR>.fchain-cert.pem
```

[Optional] Git commit
```
git add .
git commit -m "Issued RADIUS certificate"
git push
```
