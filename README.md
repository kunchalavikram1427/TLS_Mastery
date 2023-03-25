# SSL/TLS 

### TLS
Transport Layer Security(TLS) is a protocol that establishes an encrypted session between two computers/applications on the Internet, typically between a web browser and a webserver.  It is updated version to SSL.  It ensures that eavesdroppers and hackers are unable to see what you transmit which is particularly useful for private and sensitive information such as passwords, credit card numbers, and personal information. TLS encryption is used in HTTPS connections, which are secured using SSL/TLS certificates. Thus, HTTPS connections ensure that no one can snoop on your internet traffic while browsing the web or emailing your friends or family members.

### SSL/TLS Certificate
An SSL certificate is a type of digital certificate that provides authentication for a website and enables an encrypted connection. 
When a website holds an SSL certificate, a padlock icon appears on the left side of the URL address bar signifying that the connection is secure. Additionally, sites will display an HTTPS address instead of an HTTP address. 

## Creating a Self-Signed Certificate
A self-signed certificate is a certificate that's signed with its own private key. 
It can be used to encrypt data just as well as CA-signed certificates, but our users will be shown a warning that says the certificate isn't trusted.

You can create self-signed certificates using **OpenSSL**

**OpenSSL** is a handy utility to create self-signed certificates. You can use OpenSSL on all the operating systems such as Windows, MAC, and Linux flavors.
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout domain.key -out domain.crt
```
Here, **Rivest-Shamir-Adleman (RSA)** is an asymmetric encryption algorithm and **-nodes** (short for "no DES") is used if you don't want to protect your private key with a passphrase. Otherwise it will prompt you for "at least a 4 character" password.

Decode Certificate
```
openssl x509 -in domain.crt -text -noout
```

## Create Certificate Authority
Let's create a private key (rootCA.key) and a self-signed root CA certificate (rootCA.crt). This will act like our private CA
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt
```
or you can directly provide the CSR details like Country,State, Common Name etc in the command
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
Now, we have a key, we need a certificate signing request (CSR).
Lets start with a `csr.conf`(CSR Configuration) with the below contents
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
Decode Certificate and observe that SAN field is missing from certificate while it is available in CSR.
The reason is that by default OpenSSL does not copy extensions from the request to the certificate.
We can fix this by providing the extensions file
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
Install Nginx on AWS EC2 running Linux AMI
```
yum install nginx -y
systemctl enable --now nginx
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
Configure Nginx with the certificate and key we created
```
mkdir -p /etc/pki/nginx/private
cp domain.crt /etc/pki/nginx/server.crt
cp domain.key /etc/pki/nginx/private/server.key
systemctl restart nginx
```

## Adding CA in browser
```
https://support.securly.com/hc/en-us/articles/206081828-How-do-I-manually-install-the-Securly-SSL-certificate-in-Chrome-
```

## [EXTRA] Creating a Self-Signed Certificate with CSR and Private key

Create a Private key
```
openssl genrsa -out domain.key 2048
```
Create CSR
```
openssl req -nodes -key domain.key -new -out domain.csr
```
We can also create both the private key and CSR with a single command:
```
openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr
```
Let's create a self-signed certificate (domain.crt) with our existing private key and CSR
```
openssl x509 -signkey domain.key -in domain.csr -req -days 365 -out domain.crt
```
We can also create a private key and a self-signed certificate with a single command
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout domain.key -out domain.crt
```

## Links
```
https://certlogik.com/decoder/
https://stackoverflow.com/questions/43665243/invalid-self-signed-ssl-cert-subject-alternative-name-missing
https://stackoverflow.com/questions/6194236/openssl-certificate-version-3-with-subject-alternative-name
https://serverfault.com/questions/845766/generating-a-self-signed-cert-with-openssl-that-works-in-chrome-58
Free SSL Certs: https://www.cminds.com/blog/wordpress/5-best-free-ssl-certificates-secure-site/
```
