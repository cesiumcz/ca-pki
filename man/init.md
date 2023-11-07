## CA configuration, nameConstraints
```
umask 077
git clone git@github.com:cesiumcz/openssl-ca.git
```

Change remote URL to target your repository:  
```
git remote set-url origin <YOUR_REMOTE_REPO_URL>
```

```
cp openssl.cnf.templ.clean openssl.cnf.templ
```

Set path to HSM OpenSC library

```
sed -i 's#<OpenSC_LIB>#/usr/lib/libeToken.so#g' openssl.cnf.templ
```

Set distinguished name defaults in `req_distinguished_name` section
```
sed -i 's#<DN_COUNTRY>#CZ#g' openssl.cnf.templ
sed -i 's#<DN_STATE>#Královéhradecký kraj#g' openssl.cnf.templ
sed -i 's#<DN_LOCALITY>#Hradec Králové#g' openssl.cnf.templ
sed -i 's#<DN_ORG>#Example.org#g' openssl.cnf.templ
sed -i 's#<DN_ORG_IDENTIFIER>#NTRCZ-123456789#g' openssl.cnf.templ
```
Set OCSP and CRL URL addresses
```
sed -i 's#<CRL_DAYS>#90#g' openssl.cnf.templ
sed -i 's#<CAISSUERS>#http://ca.example.org#g' openssl.cnf.templ
sed -i 's#<OCSP>#http://ocsp.example.org#g' openssl.cnf.templ
sed -i 's#<CRL_ROOT>#http://crl.example.org/root.crl#g' openssl.cnf.templ
sed -i 's#<CRL_DVTLS>#http://crl.example.org/dvtls.crl#g' openssl.cnf.templ
sed -i 's#<CRL_PERSONAL>#http://crl.example.org/personal.crl#g' openssl.cnf.templ
```
Set Policy OID ([acquired before](oid.md)), CPS URI and notice text
```
sed -i 's#<POLICY_OID>#1.3.6.1.4.1.{OUR_PENNumber}.1.1.1#g' openssl.cnf.templ
sed -i 's#<CPS_URI>#https://ca.example.org#g' openssl.cnf.templ
sed -i 's#<NOTICE>#This certificate was issued by Example.org. in accordance with company certificate policy.#g' openssl.cnf.templ
sed -i 's#<NOTICE_PERSONAL>#This personal certificate was issued by Example.org. in accordance with company certificate policy to its employee for the stated purposes.#g' openssl.cnf.templ
sed -i 's#<NOTICE_DVTLS>#This SSL/TLS certificate was issued by Example.org. in accordance with company certificate policy to the employee responsible for services running on the specified domain name / IP address.#g' openssl.cnf.templ
```
Set name constraints for issuing CAs
- DNS: *.example.lan, *.example.org (including subdomains)
- Mail: *.example.org (including subdomains)
- IP _(see warnings below!)_:
    - IPv4 loopbacks + all private IPv4 ranges = 127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
    - IPv6 loopback + unique local range fc00::/7

```
vim +182 openssl.cnf.templ
[v3_dvtls_name_constraints]
...
permitted;DNS.1 = example.org
permitted;DNS.2 = .example.org
permitted;DNS.3 = example.lan
permitted;DNS.4 = .example.lan
...
[v3_personal_name_constraints]
permitted;email.1 = example.org
permitted;email.2 = .example.org
```

> **_WARNING: Issuing certificates for IP addresses or Internal Name is strongly not recommended._**  
> Because reserved IP addresses and Internal Names (intranet) are not unique, they are easy to impersonate by attackers to commit man-in-the-middle attacks and get unauthorized access to the data.  
> According to [CA/Browser forum Baseline Requirements](https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-v2.0.1.pdf):  
> - 2015-11-01 - Issuance of Certificates with Reserved IP Address or Internal Name prohibited.  
> - 2016‐10‐01 - All Certificates with Reserved IP Address or Internal Name must be revoked.  

> **_WARNING: Do not use .local as a private TLD._**  
The best practice is to use a subdomain of a public domain (corp.example.org). If we do not have any public domain, we can use non-conflicting TLDs according to [RFC 6762 Appendix G](https://www.rfc-editor.org/rfc/rfc6762#appendix-G):
> - .intranet.
> - .internal.
> - .private.
> - .corp.
> - .home.
> - .lan.
>
>> Using ".local" as a private top-level domain conflicts with Multicast DNS and may cause problems for users.

> **_NOTE on nameConstraints: SSL/TLS inspection firewall_**  
If you plan to use DV TLS CA also for SSL/TLS inspection firewall, do not apply `nameConstraints` (for obvoius reasons).  
However, typically, we would create an extra, unconstrained CA just for SSL/TLS inspection and issue certificates for firewalls.  
*This note does not relate to Personal CA, in which case `nameConstraints` for e-mail addresses are entirely relevant.*
