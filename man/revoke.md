## Revoking a certificate
```
umask 077
git pull
cd pki/<CA_OF_COMPROMISED_KEY>
```
Revoke certificate using CA keypair in HSM
```
openssl ca -config ./openssl.cnf \
      -engine pkcs11 -keyform engine \
      -keyfile 0:<CA_KEY_ID> \
      -revoke certs/<CERTNAME>.cert.pem
```
[Alternative] certificate using CA keypair in computer
```
openssl ca -config ./openssl.cnf \
      -revoke certs/<CERTNAME>.cert.pem
```
[Then issue new CRL](issue-crl.md)

In case issuing CA was revoked, its directory can be deleted since it no longer exists.

[Optional] Git commit
```
git add .
git commit -m "Revoked ... certificate."
git push
```