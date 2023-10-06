## Generating Client Revoke List
```
umask 077
git pull
cd pki/<CA_DIRECTORY>
mkdir -p crl
```

Generate new CRL using CA private key in HSM
```
openssl ca -config ./openssl.cnf \
      -engine pkcs11 -keyform engine -keyfile 0:<CA_KEY_ID> \
      -gencrl -out crl/ca.crl.pem
```
[Alternative] Generate new CRL using CA private key in computer
```
openssl ca -config ./openssl.cnf -gencrl -out crl/ca.crl.pem
```

Convert from PEM to DER for most effective network transfer
```
openssl crl -in crl/ca.crl.pem -outform DER -out crl/ca.crl.der
```

Check CRL contents
```
openssl crl -in crl/ca.crl.der -noout -text
```

**Do not forget to upload new CRL onto webserver!**

[Optional] Git commit
```
git add .
git commit -m "Issued new CRL"
git push
```
