# Httpd virtual hosts with LetsEncrypt Certificates
Add EPEL Repository:

    

   

     dnf install epel-release -y

Install certbot client, certbot apache module and httpd package:
     

   

    dnf install certbot certbot-apache httpd python pip -y
    pip install certbot-dns-cloudflare

    init 6


create directories to host website files:

    mkdir -p /var/www/vhosts/{prod,stage,test}

create the virtual hosts:

    vim /etc/httpd/conf.d/vhosts.conf 


        
    <VirtualHost *:80>
        # This is main web server.
        DocumentRoot "/var/www/html"
        ServerName Aus.tayyabtahir.net
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/prod"
        ServerName prod.aus.tayyabtahir.net
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/stage"
        ServerName stage.aus.tayyabtahir.net
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/test"
        ServerName test.aus.tayyabtahir.net
    </VirtualHost>

create the index files in the relavent document roots.

    echo '<h1>This is a Production site.</h1>' > /var/www/vhosts/prod/index.html
    echo '<h1>This is a Stagging site!</h1>' > /var/www/vhosts/stage/index.html
    echo '<H1>This is Testing site.</h1>' > /var/www/vhosts/test/index.html

    
enable and restart the httpd service:
    systemctl enable httpd --now
    systemctl restart httpd

Allow port 80 in firewalld and restart the firewalld:
    firewall-cmd --add-port=80/tcp --permanent
    firewall-cmd --reload

    curl cat.qrg.tayyabtahir.com

First do a dry run to see if everything is configured correct:
    certbot certonly  --apache  --dry-run --preferred-challenges=http

Now lets get the ssl certificate from stagging server.

**60 unsuccessful certificate requests per hour are allowed from stagging server.**

**5 unsuccessful certificate request per hour are allowed from production server.**


So, its always recommended to get the certificate from the stagging server first.

To get the stagging certificate run the following command:


    certbot run  --apache --test-cert --preferred-challenges=http    

Run the following command to see all obtained certificates:

    certbot certificates
Run the following command to delete the certificates:
   
     certbot delete

**Remember; Deleting certificates using the above command will remove the certificates from the /etc/letsencrypt/live/prod.aus.tayyabtahir.net/ location, but will not remove the certificate entries from the Apache config files. In this case, you will need to remove the /etc/httpd/conf.d/vhosts-le-ssl.conf file before continuing.**

Run the following command to delete vhosts-le-ssl.conf file:

    rm -f /etc/httpd/conf.d/vhosts-le-ssl.conf 

run the following command to get the production certificate:

    certbot run  --apache --preferred-challenges=http

certbot-apache module can automatically put the certificate in the httpd configuration and virtual host files.

    <VirtualHost *:80>
        # This is main web server.
        DocumentRoot "/var/www/html"
        ServerName aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/prod"
        ServerName prod.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =prod.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/test"
        ServerName test.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =test.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>

    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/stage"
        ServerName stage.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =stage.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>






To renew all created certificates:

    certbot renew

To rewnew certificates with cronjob, add the following command in cronjob. 

    crontab -e
    34 5 * * * /usr/bin/certbot renew --post-hook "systemctl reload httpd"

# WildCard Certificate with DNS Challenge:


To get the wildcard certificate using dns challenge, first you need to create the api token from cloudflare.
copy the token into a localfile for example: 

    vim .cloudflare.ini
    dns_cloudflare_api_token=safkhakd;fjahlsdfkhalkjfhalkjhfdljkas

set the following permissions on the file:

    chmod 600 .cloudflare.ini
  * With the DNS challenge, the "run" command isn't applicable; instead, you're limited to using the "certonly" command.

then run the following command to generate the wild card certificate:


    [root@TayyabsFedora ~]#  certbot certonly --dns-cloudflare --dns-cloudflare-credentials cloudflare.ini   -d '*.aus.tayyabtahir.net'
    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Requesting a certificate for *.aus.tayyabtahir.net
    Waiting 10 seconds for DNS changes to propagate

    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/aus.tayyabtahir.net/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/aus.tayyabtahir.net/privkey.pem
    This certificate expires on 2024-07-23.
    These files will be updated when the certificate renews.
    Certbot has set up a scheduled task to automatically renew this certificate in the background.

    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    If you like Certbot, please consider supporting our work by:
     * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
     * Donating to EFF:                    https://eff.org/donate-le
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    [root@TayyabsFedora ~]#

  * When opting for the DNS challenge to secure a wild-card SSL certificate, you'll have to configure Apache or Nginx settings manually post obtaining the certificates.
  * So we need to create the following 2 files and we need to mentioned ssl certificates path manually in vhosts-le-ssl.conf file.



   [root@webserver ~]# cat /etc/httpd/conf.d/vhosts.conf



    <VirtualHost *:80>
        #This is main web server.
        DocumentRoot "/var/www/html"
        ServerName aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    
    <VirtualHost *:80>
        DocumentRoot "/var/www/vhosts/prod"
        ServerName prod.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =prod.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    
    <VirtualHost *:80>
      DocumentRoot "/var/www/vhosts/test"
      ServerName test.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =test.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    
    <VirtualHost *:80>
      DocumentRoot "/var/www/vhosts/stage"
      ServerName stage.aus.tayyabtahir.net
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =stage.aus.tayyabtahir.net
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>






    [root@TayyabsFedora ~]# cat /etc/httpd/conf.d/vhosts-le-ssl.conf
    <IfModule mod_ssl.c>
    <VirtualHost *:443>
      DocumentRoot "/var/www/vhosts/prod"
      ServerName prod.aus.tayyabtahir.net
    
    
    SSLCertificateFile /etc/letsencrypt/live/aus.tayyabtahir.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/aus.tayyabtahir.net/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
    </VirtualHost>
    <VirtualHost *:443>
      DocumentRoot "/var/www/vhosts/stage"
      ServerName stage.aus.tayyabtahir.net
    

    SSLCertificateFile /etc/letsencrypt/live/aus.tayyabtahir.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/aus.tayyabtahir.net/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
    </VirtualHost>
    <VirtualHost *:443>
      DocumentRoot "/var/www/vhosts/test"
      ServerName test.aus.tayyabtahir.net
    
    
    SSLCertificateFile /etc/letsencrypt/live/aus.tayyabtahir.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/aus.tayyabtahir.net/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
    </VirtualHost>
    </IfModule>
    





