## HSM (SmartCard / USB token) initialization & environment setup
We will demonstrate the installation and the use of PKCS#11 on [Thales SafeNet eToken 5300 Series](https://cpl.thalesgroup.com/access-management/authenticators/pki-usb-authentication/etoken-5300-usb-token).

### Environment setup
OS must be configured to work with SafeNet eToken 5300 using `libeToken.so` library and various packages.  
Here is how to setup a Debian-like Linux system:

```
sudo apt-get install libccid opensc opensc-pkcs11 libengine-pkcs11-openssl
cd /tmp
curl -o sac.zip https://www.globalsign.com/en/safenet-drivers/USB/10.8/Safenet-Ubuntu-2004.zip
mkdir sac
unzip sac.zip -d sac
sudo dpkg -i sac/Ubuntu-2004/safenetauthenticationclient_10.8.28_amd64.deb
rm -rf sac sac.zip
```

Test OpenSSL pkcs11 engine
```
openssl engine pkcs11 -t
```

How to remove SAC package
```
sudo dpkg -r safenetauthenticationclient
```

### Token initialization
```
pkcs11-tool --module libeToken.so --init-token --label EXAMPLE-CA
```
