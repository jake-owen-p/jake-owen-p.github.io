# Generating SSL Certificates

This is based off [AWS SSL on Linux 2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/SSL-on-amazon-linux-2.html)

## Install the Apache Web Server:

```bash
sudo yum update -y
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
sudo yum install -y httpd mariadb-server
sudo systemctl start httpd
sudo systemctl enable httpd
```

## Enable TLS on ECS:

Ensure Apache server started:
```bash
sudo systemctl is-enabled httpd
sudo systemctl start httpd && sudo systemctl enable httpd
```

### Add TLS Support:

```bash
sudo yum install -y mod_ssl
```

## Obtaining a CA Certifiate (Skip to `Server Config` if this is done):

### Generating CSR Key:

```bash
CD /etc/pki/tls/private/
sudo openssl genrsa -out custom.key 4096
sudo chown root:root custom.key
sudo chmod 600 custom.key
sudo openssl req -new -key custom.key -out csr.pem
```

![Required](./needed.png)

### Verifying with CA Authority

The csr.pem that was just generated can be opened and the contents copied and pasted into the following site

Verifying ceritifcate with CA https://app.zerossl.com/ (THIS WILL CURRENTLY NEED TO RENEW EVERY 90 DAYS - WHEN IT COMES TO READ THING SHOULD LOOK AT DOING IT FOR A YEAR)

This will give you two files back, a cerficicate,crt and a ca_modules.crt

## When CA Files obtained

This need to be copied into the following directory, then secured. IF THE KEY ABOVE WAS CREATED TO ON AN EXTERNAL MACHINE IT MUST BE ADDED HERE.

Where certiciate.crt -> custom.crt
and ca_modules.crt -> intermediate.crt

```bash
cd /etc/pki/tls/certs

sudo chown root:root custom.crt
sudo chmod 600 custom.crt

sudo chown root:root intermediate.crt
sudo chmod 644 intermediate.crt

sudo chown root:root custom.key
sudo chmod 600 custom.key
```

## Server Config

Make sure the following lines are in `/etc/httpd/conf.d/ssl.conf`

```bash
SSLCertificateFile /etc/pki/tls/certs/custom.crt
SSLCACertificateFile /etc/pki/tls/certs/intermediate.crt
SSLCertificateKeyFile /etc/pki/tls/private/custom.key
```

## Routing Apache to you NodeJs App

```conf
<VirtualHost *:80>
  ProxyRequests Off

  ProxyPass / http://localhost:3000/
  ProxyPassReverse / http://localhost:3000/
</VirtualHost>

<VirtualHost *:443>
  ProxyRequests Off

  ProxyPass / http://localhost:3000/
  ProxyPassReverse / http://localhost:3000/
  ServerName prykor.com
  SSLEngine on
  SSLCertificateFile "/etc/pki/tls/certs/custom.crt"
  SSLCertificateKeyFile "/etc/pki/tls/private/custom.key"
</VirtualHost>
```

## Restart

```bash
sudo systemctl restart httpd
```
