# 1. Configure server: Nginx

Create the certificate:

    $ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

Create a strong Diffie-Hellman group:

    $ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
    
Create a new configuration snippet file for Nginx:

    $ sudo nano /etc/nginx/snippets/self-signed.conf
    
Add:

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    
Create a configuration snippet with strong encryption settings:

    $ sudo vim /etc/nginx/snippets/ssl-params.conf

Add:

    # from https://cipherli.st/
    # and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    #ssl_stapling on;
    #ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    # Disable preloading HSTS for now.  You can use the commented out header line that includes
    # the "preload" directive if you understand the implications.
    #add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    #add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    ssl_dhparam /etc/ssl/certs/dhparam.pem;

Configure Nginx site to use certificate: 

    server {

        listen 443 ssl;
        server_name example.com;
        include snippets/self-signed.conf;
        include snippets/ssl-params.conf;

        #...
    }


# 2. Configure computer: macOS

From local computer, download the certificate:

    $ scp user@host:/etc/ssl/certs/nginx-selfsigned.crt ~/cert.crt
    
Open the file with the Keychain Access utility:

    $ open cert.crt
    
1. Add the certificate to the System keychain (not login), authenticate.
2. After it has been added, double-click it, authenticate again.
3. Expand the "Trust" section.
4. Set "When using this certificate" to "Always Trust"

That's it! Close Keychain Access and restart Chrome, and your self-signed certificate should be recognized now by the browser.


Sources :
* https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-16-04
* http://www.robpeck.com/2010/10/google-chrome-mac-os-x-and-self-signed-ssl-certificates/#.WEbrS6LhB-g
