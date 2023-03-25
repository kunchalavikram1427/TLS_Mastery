# SSL/TLS 

## Links
```
https://certlogik.com/decoder/
https://stackoverflow.com/questions/43665243/invalid-self-signed-ssl-cert-subject-alternative-name-missing
https://devopscube.com/create-self-signed-certificates-openssl/
https://stackoverflow.com/questions/6194236/openssl-certificate-version-3-with-subject-alternative-name
https://serverfault.com/questions/845766/generating-a-self-signed-cert-with-openssl-that-works-in-chrome-58
Free SSL Certs: https://www.cminds.com/blog/wordpress/5-best-free-ssl-certificates-secure-site/

```

## Creating a Self-Signed Certificate
A self-signed certificate is a certificate that's signed with its own private key. 
It can be used to encrypt data just as well as CA-signed certificates, but our users will be shown a warning that says the certificate isn't trusted.

You can create self-signed certificates using OpenSSL commands

Openssl is a handy utility to create self-signed certificates. You can use OpenSSL on all the operating systems such as Windows, MAC, and Linux flavors.

The Rivest-Shamir-Adleman (RSA) encryption algorithm is an asymmetric encryption algorithm
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout domain.key -out domain.crt

-nodes (short for "no DES") if you don't want to protect your private key with a passphrase. Otherwise it will prompt you for "at least a 4 character" password.
```
Decode Certificate
```
openssl x509 -in domain.crt -text -noout
```

## Create Certificate Authority
Let's create a private key (rootCA.key) and a self-signed root CA certificate (rootCA.crt) from the command line:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt
```
or
```
openssl req -x509 \
            -sha256 \
            -days 356 \
            -nodes \
            -newkey rsa:2048 \
            -subj "/C=IN/ST=KA/L=BLR/O=DME/OU=DevSecOps/CN=www.devopsmadeeasy.in" \
            -keyout rootCA.key -out rootCA.crt 
```
Decode Certificate
```
openssl x509 -in rootCA.crt -text -noout
```

## Create Private Key and CSR
Create the Server Private Key
```
openssl genrsa -out domain.key 2048
```
Now we have a key, we need a certificate signing request (CSR).
Lets start with a csr.conf(CSR Configuration) with the below contents
```
cat > csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = req_distinguished_name

[req_distinguished_name]
C = IN
ST = Karnataka
L = Bengaluru
O = DME
OU = DevSecOps
CN = test.mynginx.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = test.mynginx.com
DNS.2 = mynginx.com
DNS.3 = www.mynginx.com 
IP.1 = 13.233.110.41

EOF
```
Generate CSR using Private Key of the domain
```
openssl req -new -key domain.key -out domain.csr -config csr.conf
```
Decode CSR and observe the SAN field. It will be present in CSR
```
openssl req -text -in domain.csr
```
Sign CSR with CA and Genereate SSL certificate
```
openssl x509 -req \
    -in domain.csr \
    -CA rootCA.crt -CAkey rootCA.key \
    -CAcreateserial -out domain.crt \
    -days 365 
```
Decode Certificate and observe that SAN is missing from certificate while it is available in CSR.
The reason is that by default OpenSSL does not copy extensions from the request to the certificate.
Normally, the certificate would be created/signed by a CA based on a request from a customer, and some extensions could grant the certificate more power than the CA was intending if they were to blindly trust the extensions defined in the request.
```
openssl x509 -in domain.crt -text -noout
```

Create a external file to include SAN in the certificate
```
cat > v3.ext <<EOF

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = test.mynginx.com
DNS.2 = mynginx.com
DNS.3 = www.mynginx.com 
IP.1 = 13.233.110.41

EOF
```
```
# Generate SSL certificate With self signed CA
openssl x509 -req \
    -in domain.csr \
    -CA rootCA.crt -CAkey rootCA.key \
    -CAcreateserial -out domain.crt \
    -days 365 \
    -extfile v3.ext
```
Decode Certificate and observe that SAN is available in the certificate
```
openssl x509 -in domain.crt -text -noout
```

## Install and configure Nginx
```
yum install nginx -y
systemctl enable --now nginx
```
Configure Nginx with the certificate and key
```
mkdir -p /etc/pki/nginx/private
cp domain.crt /etc/pki/nginx/server.crt
cp domain.key /etc/pki/nginx/private/server.key
systemctl restart nginx
```
Enable SSL by editing the configuration `/etc/nginx/nginx.conf` to listen on 443
```
server {

listen   443;

ssl    on;
ssl_certificate    /etc/ssl/server.crt;
ssl_certificate_key    /etc/ssl/server.key;

server_name your.domain.com;
access_log /var/log/nginx/nginx.vhost.access.log;
error_log /var/log/nginx/nginx.vhost.error.log;
location / {
root   /home/www/public_html/your.domain.com/public/;
index  index.html;
}

}

```

## Adding CA in browser
```
https://docs.vmware.com/en/VMware-Adapter-for-SAP-Landscape-Management/2.1.0/Installation-and-Administration-Guide-for-VLA-Administrators/GUID-D60F08AD-6E54-4959-A272-458D08B8B038.html
```

## [EXTRA] Creating a Self-Signed Certificate with CSR and Private key
```
# Create a Private key
openssl genrsa -out domain.key 2048

# Create CSR
openssl req -nodes -key domain.key -new -out domain.csr

The "challenge password" is basically a shared-secret nonce between you and the SSL certificate-issuer (aka Certification Authority, or CA), embedded in the CSR, which the issuer may use to authenticate you should that ever be needed.

# We can also create both the private key and CSR with a single command:
openssl req -newkey rsa:2048 -keyout domain.key -out domain.csr

# If we want our private key unencrypted, we can add the -nodes option:
openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr

# Let's create a self-signed certificate (domain.crt) with our existing private key and CSR:
openssl x509 -signkey domain.key -in domain.csr -req -days 365 -out domain.crt

# We can create a private key and a self-signed certificate with just a single command
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout domain.key -out domain.crt
```
