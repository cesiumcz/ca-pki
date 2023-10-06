## DV TLS Certificate Authority initialization
```
umask 077
cd pki
mkdir dvtls
cd dvtls
mkdir certs crl csr newcerts private
touch index.txt
openssl rand -hex 16 > serial
echo 01 > crlnumber
cp ../openssl.cnf.templ ./openssl.cnf
sed -i 's#<CA_NAME>#DV TLS CA#g' openssl.cnf
sed -i 's#<DEFAULT_DAYS>#730#g' openssl.cnf
```
Decide which policy to embed in issuing CA.  
- `policy_loose` allows arbitrary `countryName`, `stateOrProvinceName`, `localityName`, `organizationName` and `organizationIdentifier`
- `policy_strict` requires `countryName`, `stateOrProvinceName`, `localityName`, `organizationName` and `organizationIdentifier` to match Root CA

```
sed -i 's#<POLICY>#policy_strict#g' openssl.cnf
```

Generate ECC NIST-384 key in HSM with ID 2
```
pkcs11-tool --module libeToken.so \
	--login --keypairgen \
	--id 2 --label dvtls-ecc-g1 \
	--key-type EC:secp384r1
```
[Alternative] Generate key in computer and import it into HSM

```
openssl ecparam -genkey -name secp384r1 | openssl ec -aes256 -out private/ca.key.pem && chmod 400 private/ca.key.pem
```

`pkcs11-tool` needs DER encoded private key to import:
```
openssl ec -inform PEM -in private/ca.key.pem -outform DER -out private/ca.key.der
pkcs11-tool --module libeToken.so --login --write-object private/ca.key.der --type privkey --id 2 --label dvtls-ecc-g1
```
Generate request using private key in HSM
```
openssl req -config ./openssl.cnf -new \
      -engine pkcs11 -keyform engine -key 0:02 \
      -out csr/ca.csr.pem
```
[Alternative] Generate request using private key in computer
```
openssl req -config ./openssl.cnf -new \
      -key private/ca.key.pem \
      -out csr/ca.csr.pem
```
Sign dvtls certificate by root CA

```
cd ../root
```

Set name constraints for DV TLS issuing CA:

```
sed -i '/^\[v3_issuing_ca]/,/^\[/{s/^#*nameConstraints[[:space:]]*=.*/nameConstraints = critical, @v3_dvtls_name_constraints/}' openssl.cnf
```

Sign the request using Root CA in HSM
```
openssl ca -config ./openssl.cnf \
      -extensions v3_issuing_ca -days 3650 -notext \
      -engine pkcs11 -keyform engine -keyfile 0:01 \
      -in ../dvtls/csr/ca.csr.pem -out ../dvtls/certs/ca.cert.pem
```
[Alternative] Sign the request using Root CA in computer
```
openssl ca -config ./openssl.cnf \
      -extensions v3_issuing_ca -days 3650 -notext \
      -in ../dvtls/csr/ca.csr.pem \
      -out ../dvtls/certs/ca.cert.pem
```
Root CA must have DV TLS CA certificate in `certs/` for future revoke. Create a symlink to it:
```
ln -s ../../dvtls/certs/ca.cert.pem certs/dvtls.cert.pem
```
Verify the certificate
```
openssl x509 -noout -text -nameopt utf8 -in ../dvtls/certs/ca.cert.pem
```
Verify dvtls cert against root cert
```
openssl verify -CAfile certs/ca.cert.pem ../dvtls/certs/ca.cert.pem
```
Create fullchain certificate
```
cat ../dvtls/certs/ca.cert.pem certs/ca.cert.pem > ../dvtls/certs/ca.fchain-cert.pem
```

[Optional] Import the certificate into HSM
```
openssl x509 -in ../dvtls/certs/ca.cert.pem -out ../dvtls/certs/ca.cert.der -outform DER
pkcs11-tool --module libeToken.so --login --write-object ../dvtls/certs/ca.cert.der --type cert --id 2 --label dvtls-ecc-g1
```
[Optional] Git commit
```
git add .
git commit -m "Created DV TLS CA (HSM ID=2)"
git push
```
Then:
- [Generate CRL](issue-crl.md)
- [Issue OCSP certificate](issue-ocsp.md)
