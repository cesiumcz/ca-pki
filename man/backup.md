## Private key backup
The following text shows how to create an encrypted backup of CA private keys (and certificates).  
The keys are encrypted by asymetrically encrypted AES symmetric cipher.
The following text assumes USB drive mounted on `/mnt/usb/`.

### Creating backup
Simplified diagram. Highlighted files are backuped.
```
Generate asymmetric keys
                                                 openssl genrsa <---- PASSWORD
                                                       |                  |
                                                       V                  |
                                                  usb.key.pem*            |
                                                       |                  V
                                                       +-------> openssl rsa -pubout
                                                       |                  |
                                                       |                  V
                                                       |             usb.pub.pem
Generate symmetric key & encrypt                       |                  |
                                                       V                  V         ==============
                  /dev/random ----> aeskey.bin ----> openssl pkeyutl -encrypt ----> aeskey.enc.bin
                                         |                                          ==============
Encrypt .tar using AES key               |
                                         V            ===============
ca-keys/ ----> ca-keys.tar ----> openssl enc -e ----> ca-keys.enc.tar
                      |                               ===============
Create a signature    |
                      V                     ===================
*usb.key.pem ----> openssl dgst -sign ----> ca-keys.tar.sig.bin
                                            ===================
```
#### Generate keys to encrypt USB drive
```
umask 077
mkdir /tmp/ca
cd /tmp/ca
openssl genrsa -aes256 -out usb.key.pem 4096
openssl rsa -in usb.key.pem -outform PEM -pubout -out usb.pub.pem
openssl rand 256 > aeskey.bin
```

#### Create tar archive, encrypt and sign
**Assume keys are already in `ca-keys` directory.**
```
cd /tmp/ca
tar cvf ca-keys.tar ca-keys/
openssl enc -e -aes256 -salt -pbkdf2 -pass file:./aeskey.bin -in ca-keys.tar \
    | openssl base64 -out ca-keys.tar.enc.b64
```
Encrypt the key
```
openssl pkeyutl -encrypt -inkey usb.pub.pem -pubin -in aeskey.bin -out aeskey.enc.bin
openssl base64 -in aeskey.enc.bin -out aeskey.enc.b64
```
Create a signature
```
openssl dgst -sha512 -sign usb.key.pem -out ca-keys.tar.sig.bin ca-keys.tar
openssl base64 -in ca-keys.tar.sig.bin -out ca-keys.tar.sig.b64
tar -cvf ca-keys.enc.tar aeskey.enc.b64 ca-keys.tar.enc.b64 ca-keys.tar.sig.b64
```
Move ca-keys.enc.tar onto USB drive
```
mv ca-keys.enc.tar /mnt/usb/
```
**Do not forget to save usb.key.pem and usb.pub.pem to Keepass (for example).**

### Restoring backup
#### Diagram
Simplified diagram. Highlighted file is decrypted archive.
```
Decrypt AES key      usb.key.pem    PASSWORD
                          |            |
                          V            V
aeskey.enc.bin ----> openssl pkeyutl -decrypt ----> aeskey.bin
                                                        |
Decrypt archive                                         V
                            ca-keys.tar.enc ----> openssl enc -d
                                                        |
                                                        V
                                                   ===========
                                                   ca-keys.tar
                                                   ===========
                                                        |
Verify signature                                        V
                             usb.pub.pem ----> openssl dgst -verify <---- ca-keys.tar.sig.bin
                                                        |
                                                        V
                                                    OK / FAIL
```
#### Procedure
```
umask 077
tar xf ca-keys.enc.tar && rm ca-keys.enc.tar
```
Decode Base64 encoded files first
```
openssl base64 -d -in aeskey.enc.b64 -out aeskey.enc.bin
openssl base64 -d -in ca-keys.tar.enc.b64 -out ca-keys.tar.enc.bin
openssl base64 -d -in ca-keys.tar.sig.b64 -out ca-keys.tar.sig.bin
rm *.b64
```
Asymetrically decrypt AES key
```
openssl pkeyutl -decrypt -inkey usb.key.pem -in aeskey.enc.bin -out aeskey.bin && rm aeskey.enc.bin
```
Symetrically decrypt encrypted tar archive
```
openssl enc -d -aes-256-cbc -pbkdf2 -in ca-keys.tar.enc.bin -out ca-keys.tar -pass file:./aeskey.bin
```
Verify the signature
```
openssl dgst -sha512 -verify usb.pub.pem -signature ca-keys.tar.sig.bin ca-keys.tar
```
If it returned "Verified OK", extract the private keys:
```
tar xf ca-keys.tar
```
