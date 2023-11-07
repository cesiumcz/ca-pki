## Issuing DV TLS server certificate
### Host device
On host device, generate the key and request using `openssl.cnf` from `pki/dvtls/openssl.cnf`
```
umask 077
git pull
cd pki/dvtls
mkdir -p csr private newcerts
```
Generate private key
```
# NIST P-256
openssl ecparam -genkey -name prime256v1 | openssl ec -aes256 -out private/<HOST>.example.lan_<YEAR>.key.pem && chmod 400 private/<HOST>.example.lan_<YEAR>.key.pem
# RSA 2048 bit
openssl genrsa -aes256 -out private/<HOST>.example.lan_<YEAR>.key.pem 2048 && chmod 400 private/<HOST>.example.lan_<YEAR>.key.pem
```
Set alternative name(s)  
**WARNING: Issuing certificates for IP addresses or Internal Name is strongly not recommended.**
```
vim +149 openssl.cnf
```
```
[v3_server_alt_names]
DNS.1                  = <HOST>.example.lan
#DNS.2                  = localhost
#IP.1                   = 127.0.0.1
#IP.2                   = <HOST_IP>
```

Create a request. Set CN to <HOST>.example.lan, leave email blank
```
openssl req -config ./openssl.cnf -extensions v3_server -new \
	-key private/<HOST>.example.lan_<YEAR>.key.pem \
	-outform DER -out csr/<HOST>.example.lan_<YEAR>.csr.der
```

Transfer <HOST>.example.lan_<YEAR>.csr.der to signing machine (pki/dvtls/csr/)

### Signing device
```
cd pki/dvtls
```

**Check and verify all distinguished name fields**
```
openssl req -noout -text -nameopt utf8 -inform DER -in csr/<HOST>.example.lan_<YEAR>.csr.der
```

> **_NOTE:_** x509v3 extensions are intentionally not copied automatically to the certificate (`-copy_extensions`) due to high risk of introducing dangerous values (CA:TRUE / keyUsage).  
Thus, extended attributes (like e-mail addresses, DNS names or IP addresses) must be manually verified and rewritten by man.  
In case of wrong request DN, instead of rejecting the request and asking the applicant to regenerate teh request, we can fix DN by using `-subj` argument for `openssl ca`:  
`-subj "/C=<Country Name>/ST=<State>/L=<Locality Name>/O=<Organization Name>/CN=<Common Name>/"`  
Fields that are set to match according to policy does not need to be fixed since they are copied from signing certificate.

Set alternative name(s)  
**WARNING: Issuing certificates for IP addresses or Internal Name is strongly not recommended.**
```
vim +149 openssl.cnf
```
[v3_server_alt_names]
DNS.1                  = <HOST>.example.lan
#DNS.2                  = localhost
#IP.1                   = 127.0.0.1
#IP.2                   = <HOST_IP>
```

Sign the request using DV TLS CA private key in HSM.
```
openssl ca -config ./openssl.cnf \
	-extensions v3_server -days 730 -notext \
	-engine pkcs11 -keyform engine -keyfile 0:02 \
	-inform DER -in csr/<HOST>.example.lan_<YEAR>.csr.der -out certs/<HOST>.example.lan_<YEAR>.cert.pem
```
[Alternative] Sign the request using DV TLS CA private key in computer
```
openssl ca -config ./openssl.cnf \
	-extensions v3_server -days 730 -notext \
	-inform DER -in csr/<HOST>.example.lan_<YEAR>.csr.der -out certs/<HOST>.example.lan_<YEAR>.cert.pem
```

Undo changes in `openssl.cnf`
```
git checkout HEAD -- openssl.cnf
```

Verify the certificate
```
openssl x509 -noout -text -nameopt utf8 -in certs/<HOST>.example.lan_<YEAR>.cert.pem
openssl verify -CAfile certs/ca.fchain-cert.pem certs/<HOST>.example.lan_<YEAR>.cert.pem
```

Create fullchain certificate
```
cat certs/<HOST>.example.lan_<YEAR>.cert.pem certs/ca.fchain-cert.pem > certs/<HOST>.example.lan_<YEAR>.fchain-cert.pem
```

Create PKCS#12 bundle
```
openssl pkcs12 -export -out private/<HOST>.example.lan_<YEAR>.p12 -inkey private/<HOST>.example.lan_<YEAR>.key.pem -in certs/<HOST>.example.lan_<YEAR>.fchain-cert.pem
```
[Optional] Git commit
```
git add .
git commit -m "Issued DV TLS certificate for ..."
git push
```
