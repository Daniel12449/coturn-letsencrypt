# Setup coturn on debian 11 with Let's Encrypt behind NAT

## Install Coturn & Certbot
```
sudo apt-get install coturn snapd
```
## Setup Certbot [Based on Certbot documentation](https://certbot.eff.org/lets-encrypt/debianbuster-other).  
I'm using DNS-01 challenge with cloudflare, other options available in certbot docs.
```
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo snap set certbot trust-plugin-with-root=ok
sudo snap install certbot-dns-cloudflare
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
Create the credentials file:
```
sudo mkdir /root/.secrets
sudo cat > /root/.secrets/cloudflare.ini << EOF
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
EOF
sudo chmod 600 /root/.secrets/cloudflare.ini
```
To test if certbot is installed correctly and let it create its folder structure, run `sudo certbot`.

And create a renew-hook script to let coturn access the certs without root privileges.  
This script is a modified version of [jitsi´s certbot depoloy script](https://github.com/jitsi/jitsi-meet/blob/master/doc/debian/jitsi-meet-turn/coturn-certbot-deploy.sh)

```
sudo nano /etc/letsencrypt/renewal-hooks/post/example.com.conf
```
```
#!/bin/sh

set -e

COTURN_CERT_DIR="/etc/coturn/certs"
TURN_CONFIG="/etc/turnserver.conf"
domain="example.com"

# create a directory to store certs if it does not exists
if [ ! -d "$COTURN_CERT_DIR" ]; then
    mkdir -p $COTURN_CERT_DIR
    chown -R turnserver:turnserver /etc/coturn/
    chmod -R 700 /etc/coturn/
fi

# This is a template and when copied to /etc/letsencrypt/renewal-hooks/deploy/
# during creating the Let's encrypt certs script
# jitsi-meet.example.com will be replaced with the real domain of deployment

# Make sure the certificate and private key files are
# never world readable, even just for an instant while
# we're copying them into daemon_cert_root.
umask 077

cp "/etc/letsencrypt/live/$domain/fullchain.pem" "$COTURN_CERT_DIR/$domain.fullchain.pem"
cp "/etc/letsencrypt/live/$domain/privkey.pem" "$COTURN_CERT_DIR/$domain.privkey.pem"

# Apply the proper file ownership and permissions for
# the daemon to read its certificate and key.
chown turnserver "$COTURN_CERT_DIR/$domain.fullchain.pem" \
        "$COTURN_CERT_DIR/$domain.privkey.pem"
chmod 400 "$COTURN_CERT_DIR/$domain.fullchain.pem" \
        "$COTURN_CERT_DIR/$domain.privkey.pem"

if [ -f $TURN_CONFIG ] ; then
    echo "Configuring turnserver"
    sed -i "/^cert/c\cert=\/etc\/coturn\/certs\/${domain}.fullchain.pem" $TURN_CONFIG
    sed -i "/^pkey/c\pkey=\/etc\/coturn\/certs\/${domain}.privkey.pem" $TURN_CONFIG
    fi
#service coturn restart
```
Make the deploy script executable
```
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/example.com.conf
```
And finally request your certificate
```
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d example.com
```
You may need to run `sudo certbot renew --dry-run`to actually trigger the script!
## Setting up coturn as a Service
To enable coturn at startup, uncomment this line in `/etc/default/coturn`:
```
TURNSERVER_ENABLED=1
```

Replace the config in `/etc/turnserver.conf` or adjust the existing File.
```
realm=example.com
server-name=example.com

fingerprint

listening-ip=internal-ip
external-ip=external-ip/internal-ip


listening-port=3478
tls-listening-port=5349
no-tlsv1
no-tlsv1_1

min-port=49160
max-port=49200

log-file=/var/log/coturn/turnserver.log

verbose

# TEST-USER DISABLE AFTER TESTING
user=test:test

lt-cred-mech
use-auth-secret

# generate secret with pwgen -s 64 1
static-auth-secret=

cipher-list="ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384"
cert=/etc/coturn/certs/example.com.fullchain.pem
pkey=/etc/coturn/certs/example.com.privkey.pem
```
To allow coturn to write logs as an unprivileged user, create this directory and adjust its owner:
```
sudo mkdir /var/log/coturn
sudo chown -R turnserver:turnserver /var/log/coturn
```
Adjust the coturn systemd service to allow binding to ports:
Updates to the coturn package will overwrite its service file, so use an override file.
```
sudo mkdir /etc/systemd/system/coturn.service.d
sudo nano /etc/systemd/system/coturn.service.d/override.conf 
````
and insert:
```
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

Reload and restart coturn:
```
sudo systemctl daemon-reload
sudo systemctl restart coturn.service
```

## Forward Ports to coturn

Coturn as configured needs following ports:
```
3478 udp/tcp
5349 udp/tcp
49160-49200 udp
```

Example iptables rules to forward Ports to coturn vm:
```
iptables -t nat -A PREROUTING -i enp5s0 -p udp --dport 3478 -j DNAT --to coturn-local-ip:3478
iptables -t nat -A PREROUTING -i enp5s0 -p tcp --dport 3478 -j DNAT --to coturn-local-ip:3478
iptables -t nat -A PREROUTING -i enp5s0 -p udp --dport 5349 -j DNAT --to coturn-local-ip:5349
iptables -t nat -A PREROUTING -i enp5s0 -p tcp --dport 5349 -j DNAT --to coturn-local-ip:5349
iptables -t nat -A PREROUTING -i enp5s0 -p udp --dport 49160:49200 -j DNAT --to coturn-local-ip:49160-49200
```

