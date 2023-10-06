## Root Certificate Authority initialization
Create directory structure
```
umask 077
cd pki
mkdir root
cd root
mkdir certs crl csr newcerts private
touch index.txt
openssl rand -hex 16 > serial
echo 01 > crlnumber
cp ../openssl.cnf.templ ./openssl.cnf
sed -i 's#<CA_NAME>#Root CA#g' openssl.cnf
sed -i 's#<DEFAULT_DAYS>#3650#g' openssl.cnf
sed -i 's#<POLICY>#policy_strict#g' openssl.cnf
```

Generate ECC NIST-384 key in HSM with ID 1
```
pkcs11-tool --module libeToken.so \
	--login --keypairgen \
	--id 1 --label root-ecc-g1 \
	--key-type EC:secp384r1
```
[Alternative] Generate key in computer and import it into HSM
```
openssl ecparam -genkey -name secp384r1 | openssl ec -aes256 -out private/ca.key.pem && chmod 400 private/ca.key.pem
```

`pkcs11-tool` needs DER encoded private key to import:
```
openssl ec -inform PEM -in private/ca.key.pem -outform DER -out private/ca.key.der
pkcs11-tool --module libeToken.so --login --write-object private/ca.key.der --type privkey --id 1 --label root-ecc-g1
```
Create certificate using private key in HSM, slot 0, key ID 01. `-key` accepts RFC7512 URI scheme
```
openssl req -config ./openssl.cnf \
      -new -x509 -days 7300 -extensions v3_ca \
      -engine pkcs11 -keyform engine -key 0:01 \
      -out certs/ca.cert.pem
```
[Alternative] Create certificate using private key in computer
```
openssl req -config ./openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -extensions v3_ca \
      -out certs/ca.cert.pem
```

Verify the root certificate
```
openssl x509 -noout -text -nameopt utf8 -in certs/ca.cert.pem
```

[Optional] Import the certificate into HSM
```
openssl x509 -in certs/ca.cert.pem -out certs/ca.cert.der -outform DER
pkcs11-tool --module libeToken.so --login --write-object certs/ca.cert.der --type cert --id 1 --label root-ecc-g1
```
[Optional] Git commit
```
git add .
git commit -m "Created Root CA (HSM ID=1)"
git push
```
Then:
- [Generate CRL](issue-crl.md)
- [Issue OCSP certificate](issue-ocsp.md)
