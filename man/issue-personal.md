## Issuing Personal certificate
### Client device
On client device, generate the key and request using `openssl.cnf` from `pki/personal/openssl.cnf`
```
umask 077
git pull
cd pki/personal
mkdir -p csr private newcerts
```
Generate private key
```
# NIST P-256
openssl ecparam -genkey -name prime256v1 | openssl ec -aes256 -out private/<USERNAME>_<YEAR>.key.pem && chmod 400 private/<USERNAME>_<YEAR>.key.pem
# RSA 2048 bit
openssl genrsa -aes256 -out private/<USERNAME>_<YEAR>.key.pem 2048 && chmod 400 private/<USERNAME>_<YEAR>.key.pem
```
Set email address(es) and [optionally] Windows UPN (username) for smartcard logon in `openssl.cnf`
```
vim +126 openssl.cnf
```
[v3_personal_alt_names]
email.1                = copy
#email.2               = secondary@example.org
#email.3               = ternary@example.org
otherName.0            = 1.3.6.1.4.1.311.20.2.3;FORMAT:UTF8,UTF8:username@example.lan
```
Create a request
```
openssl req -new -config ./openssl.cnf -extensions v3_personal \
	-key private/<USERNAME>_<YEAR>.key.pem \
	-outform DER -out csr/<USERNAME>_<YEAR>.csr.der
```

Transfer `<USERNAME>_<YEAR>.csr.der` to signing machine (`pki/personal/csr/`)

### Signing device
```
cd pki/personal
```

**Check and verify all distinguished name fields**
```
openssl req -noout -text -nameopt utf8 -inform DER -in csr/<USERNAME>_<YEAR>.csr.der
```

> **_NOTE:_**  x509v3 extensions are intentionally not copied automatically to the certificate (`-copy_extensions`) due to high risk of introducing dangerous values (CA:TRUE / keyUsage).  
Thus, extended attributes (like e-mail addresses, DNS names or IP addresses) must be manually verified and rewritten by man.  
In case of wrong request DN, instead of rejecting the request and asking the applicant to regenerate teh request, we can fix DN by using `-subj` argument for `openssl ca`:  
`-subj "/C=<Country Name>/ST=<State>/L=<Locality Name>/O=<Organization Name>/CN=<Common Name>/E=<Email>"`  
Fields that are set to match according to policy does not need to be fixed since they are copied from signing certificate.

Set email address(es) and [optionally] Windows UPN (username) for smartcard logon in `openssl.cnf`
```
vim +126 openssl.cnf
```
[v3_personal_alt_names]
email.1                = copy
#email.2               = secondary@example.org
#email.3               = ternary@example.org
otherName.0            = 1.3.6.1.4.1.311.20.2.3;FORMAT:UTF8,UTF8:username@example.lan
```

Sign the request using Personal CA private key in HSM
```
openssl ca -config ./openssl.cnf \
      -extensions v3_personal -days 730 -notext \
      -engine pkcs11 -keyform engine -keyfile 0:03 \
      -inform DER -in csr/<USERNAME>_<YEAR>.csr.der -out certs/<USERNAME>_<YEAR>.cert.pem
```
[Alternative] Sign the request using Personal CA private key in computer
```
openssl ca -config ./openssl.cnf \
      -extensions v3_personal -days 730 -notext \
      -inform DER -in csr/<USERNAME>_<YEAR>.csr.der -out certs/<USERNAME>_<YEAR>.cert.pem
```

Undo changes in `openssl.cnf`
```
git checkout HEAD -- openssl.cnf
```

Verify the certificate
```
openssl x509 -noout -text -nameopt utf8 -in certs/<USERNAME>_<YEAR>.cert.pem
openssl verify -CAfile certs/ca.fchain-cert.pem certs/<USERNAME>_<YEAR>.cert.pem
```

Create fullchain certificate
```
cat certs/<USERNAME>_<YEAR>.cert.pem certs/ca.fchain-cert.pem > certs/<USERNAME>_<YEAR>.fchain-cert.pem
```

Create PKCS#12 bundle
```
openssl pkcs12 -export -out private/<USERNAME>_<YEAR>.p12 -inkey private/<USERNAME>_<YEAR>.key.pem -in certs/<USERNAME>_<YEAR>.fchain-cert.pem
```

[Optional] Git commit
```
git add .
git commit -m "Issued new Personal certificate for <USERNAME>_<YEAR>."
git push
```
