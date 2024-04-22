---
sidebar_position: 18
---
# 使用 Tongsuo 签发 SM2 双证书
## SM2测试双证书
也可以直接使用 Tongsuo 项目中已经签发的SM2双证书，仅用于测试；
根CA > 中间CA > 客户端/服务端证书，根CA证书签发中间CA证书，中间CA证书签发客户端和服务端的证书，包括签名证书和加密证书，这些证书的公钥算法都是SM2。

服务端签名证书和私钥：
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_sign.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_sign.crt)
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_sign.key](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_sign.key)

服务端加密证书和私钥：
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_enc.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_enc.crt)
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_enc.key](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/server_enc.key)

CA证书，这里面包含根CA和中间CA证书：
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/chain-ca.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/chain-ca.crt)

客户端签名证书和私钥：
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_sign.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_sign.crt)
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_sign.key](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_sign.key)

客户端加密证书和私钥：
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_enc.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_enc.crt)
- [https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_enc.key](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2/client_enc.key)

## mkcert.sh

```bash title="mkcert.sh"
export PATH=/opt/tongsuo/bin:$PATH

rm -f *.key *.csr *.crt
rm -rf {newcerts,db,private,crl}
mkdir {newcerts,db,private,crl}
touch db/{index,serial}
echo 00 > db/serial

# sm2 ca
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out ca.key

openssl req -config ca.cnf -new -key ca.key -out ca.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=root ca"

openssl ca -selfsign -config ca.cnf -in ca.csr -keyfile ca.key -extensions v3_ca -days 3650 -notext -out ca.crt -md sm3 -batch

# sm2 middle ca
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out subca.key

openssl req -config ca.cnf -new -key subca.key -out subca.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=sub ca"

openssl ca -config ca.cnf -extensions v3_intermediate_ca -days 3650 -in subca.csr -notext -out subca.crt -md sm3 -batch

cat ca.crt subca.crt > chain-ca.crt

# server sm2 double certs
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out server_sign.key

openssl req -config subca.cnf -key server_sign.key -new -out server_sign.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=server sign"

openssl ca -config subca.cnf -extensions server_sign_req -days 3650 -in server_sign.csr -notext -out server_sign.crt -md sm3 -batch

openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out server_enc.key

openssl req -config subca.cnf -key server_enc.key -new -out server_enc.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=server enc"

openssl ca -config subca.cnf -extensions server_enc_req -days 3650 -in server_enc.csr -notext -out server_enc.crt -md sm3 -batch

# client sm2 double certs
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out client_sign.key

openssl req -config subca.cnf -key client_sign.key -new -out client_sign.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=client sign"

openssl ca -config subca.cnf -extensions client_sign_req -days 3650 -in client_sign.csr -notext -out client_sign.crt -md sm3 -batch

openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:sm2 -out client_enc.key

openssl req -config subca.cnf -key client_enc.key -new -out client_enc.csr -sm3 -nodes -subj "/C=AA/ST=BB/O=CC/OU=DD/CN=client enc"

openssl ca -config subca.cnf -extensions client_enc_req -days 3650 -in client_enc.csr -notext -out client_enc.crt -md sm3 -batch
```
## ca.cnf
```bash title="ca.cnf"
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/ca.key
certificate       = $dir/ca.crt

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

req_extensions = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```
## subca.cnf
```bash title="subca.cnf"
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/subca.key
certificate       = $dir/subca.crt

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sm3

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

req_extensions = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

[ v3_req ]

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

[ server_sign_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature
# Add more items here, for instance:
# subjectAltName = @alt_names

[ server_enc_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = keyAgreement, keyEncipherment, dataEncipherment
# Add more items here, for instance:
# subjectAltName = @alt_names

[ client_sign_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature

[ client_enc_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = keyAgreement, keyEncipherment, dataEncipherment
```
