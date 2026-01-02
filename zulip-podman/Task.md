# Zulip Setup with Podman

This guide contains steps for setting up Zulip chat server using Podman and configure Postfix to use Gmail as a mail relay.

## 1. System Setup

### Update System Packages

```sh
sudo dnf update -y
```

### Add EPEL repo

```sh
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
```

### Install Required Dependencies

```sh
sudo dnf install git firewalld podman postfix python3-pip nginx -y
```

### Install Podman Compose

```sh
pip3 install podman-compose
```

Verify installation:

```sh
podman compose --version
```

### Start and Enable Nginx

```sh
sudo systemctl enable --now firewalld
sudo systemctl enable --now nginx
sudo systemctl enable --now postfix
```

## 2. Clone Zulip Docker Repository

### Download Zulip Docker Setup

```sh
cd ~
git clone https://github.com/zulip/docker-zulip.git
cd docker-zulip
```

## 3. Configure Docker Compose

### Edit Compose Configuration

```sh
vi docker-compose.yml
```

### Update Environment Variables

Replace all `REPLACE_WITH_SECURE_...` placeholders with strong passwords:

```yaml
environment:
  POSTGRES_PASSWORD: "VeryStrongPostgresPassword123!"
  SECRETS_rabbitmq_password: "VeryStrongRabbitPassword!"
  SECRETS_postgres_password: "VeryStrongPostgresPassword123!"
  SECRETS_memcached_password: "MemcachePass!"
  SECRETS_redis_password: "RedisPass!"
  SECRETS_secret_key: "SomeVeryLongRandomDjangoSecretKey"
```

Generate strong passwords using:

```sh
openssl rand -base64 32
```

### Configure External Host and email host

```yaml
SETTING_EXTERNAL_HOST: "your.domain.com"
SETTING_EMAIL_HOST = "your_private_address"
SETTING_EMAIL_PORT = "25"
SETTING_EMAIL_USE_TLS = "False"
```

Replace with your actual domain and email address.

### Modify Port Mappings

Change default ports to avoid conflicts:

```yaml
ports:
  - "1025:25" # SMTP
  - "8080:80" # HTTP
  - "8443:443" # HTTPS
```

This maps:

- Container port 25 to host port 1025 (SMTP)
- Container port 80 to host port 8080 (HTTP)
- Container port 443 to host port 8443 (HTTPS)

## 4. Configure System Limits

### Increase File Descriptor Limits

Zulip requires higher file descriptor limits for optimal performance.

Edit system limits configuration:

```sh
sudo vi /etc/security/limits.conf
```

Add the following lines (replace `youruser` with your actual username):

```conf
youruser soft nofile 1048576
youruser hard nofile 1048576
youruser soft nproc  65535
youruser hard nproc  65535
```

### Apply Limits

Log out and log back in for changes to take effect.

Verify limits:

```sh
ulimit -n
ulimit -Hn
```

Expected output should show 1048576 for file descriptors.

## 5. Configure Firewall

### Open Required Ports

```sh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --add-port=8443/tcp --permanent
sudo firewall-cmd --reload
```

### Verify Firewall Rules

```sh
sudo firewall-cmd --list-ports
```

## 6. Start Zulip Containers

### Launch Containers

```sh
cd ~/docker-zulip
podman compose up -d
```

### Verify Containers Are Running

```sh
podman ps
```

You should see containers:

```sh
$ podman ps
CONTAINER ID  IMAGE                                COMMAND               CREATED         STATUS         PORTS                                                              NAMES
a5ebc6092348  docker.io/zulip/zulip-postgresql:14  postgres              35 minutes ago  Up 35 minutes  5432/tcp                                                           docker-zulip_database_1
e88bcf920088  docker.io/library/memcached:alpine   sh -euc echo 'mec...  35 minutes ago  Up 35 minutes  11211/tcp                                                          docker-zulip_memcached_1
4ab864d48272  docker.io/library/rabbitmq:4.1       rabbitmq-server       35 minutes ago  Up 35 minutes  4369/tcp, 5671-5672/tcp, 15691-15692/tcp, 25672/tcp                docker-zulip_rabbitmq_1
b8997def1398  docker.io/library/redis:alpine       sh -euc echo "req...  35 minutes ago  Up 35 minutes  6379/tcp                                                           docker-zulip_redis_1
6b11447149cd  localhost/zulip/docker-zulip:11.4-0  app:run               35 minutes ago  Up 35 minutes  0.0.0.0:1025->25/tcp, 0.0.0.0:8080->80/tcp, 0.0.0.0:8443->443/tcp  docker-zulip_zulip_1

```

### Check Container Logs

```sh
podman compose logs -f
```

Press Ctrl+C to exit logs.

## 7. SSL/TLS Certificate Setup

### Install Certbot

```sh
sudo dnf install certbot python3-certbot-nginx -y
```

Create simple nginx configuration `/etc/nginx/conf.d/zulip.conf`

```cf
server {
    listen 80;
    listen [::]:80;
    server_name your_doman;
}
```

### Obtain SSL Certificate

```sh
sudo certbot --nginx -d your.domain.com
```

Replace `your.domain.com` with your actual domain.

## 8. Configure Nginx Reverse Proxy

### Create Nginx Configuration

```sh
sudo vi /etc/nginx/conf.d/zulip.conf
```

Add the following in https block

```nginx
...
server {
    ...other conf
    location / {
        proxy_pass https://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

```sh
sudo nginx -t
sudo systemctl reload nginx
# allow Nginx to connect to network services
sudo setsebool -P httpd_can_network_connect 1
```

## 9. Generate Realm Creation Link

### Create Organization Setup Link

```sh
podman exec -it docker-zulip_zulip_1 sudo -u zulip /home/zulip/deployments/current/manage.py generate_realm_creation_link
```

## 10. Access Zulip

### Open Browser

Navigate to:

```
https://your.domain.com
```

### Create Organization

Use the realm creation link generated earlier to set up your organization.

## 11. Configure Postfix for Email Delivery

```sh
sudo dnf install postfix mailx cyrus-sasl cyrus-sasl-plain
```

Create a app password on your google account and create file `/etc/postfix/sasl_passwd` and add

```cf
[smtp.gmail.com]:587 username@gmail.com:password
```

Edit these lines in `/etc/postfix/main.cf`

```cf
myhostname = your_domain
mydestination = localhost
relayhost = [smtp.gmail.com]:587
```

Add these lines in /etc/postfix/main.cf

```cf
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options =
smtp_sasl_password_maps = lmdb:/etc/postfix/sasl_passwd
```

### Process password file

Use **postmap** to compile and hash the contents of **sasl_passwd**. The results will be stored in your Postfix configuration directory in the file **sasl_passwd.lmdb**.

```sh
sudo postmap /etc/postfix/sasl_passwd
```

Restart postfix

```
sudo systemctl restart postfix
```

## 12. Allow Zulip Container to Relay via Host Postfix

By default, Postfix allowed only local mail from `127.0.0.1`.  
Zulip is running **inside rootless Podman** with an IP like:

`10.89.0.6`

And traffic appears on host as:

Eg. `192.168.56.17`

### Update Postfix `mynetworks`

`sudo postconf -e "mynetworks = 127.0.0.0/8, [::1]/128, 10.89.0.0/24, 192.168.56.17/32"`

### Enable relaying for trusted networks

`sudo postconf -e "smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, defer_unauth_destination"`

Restart Postfix:

`sudo systemctl restart postfix`

## 13. Testing Zulip Email Pipeline

Run the Zulip test email command:

`podman exec -it docker-zulip_zulip_1 \     sudo -u zulip /home/zulip/deployments/current/manage.py send_test_email example@gmail.com`

Expected output:

`Successfully sent 2 emails to example@gmail.com!`

Check Postfix logs:

`sudo journalctl -u postfix -f`

You should see Gmail accepting the mail:

`status=sent (250 2.0.0 OK ...)`
