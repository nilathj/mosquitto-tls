# Setup subdomains and letsencrypt certificates

These instructions will allow you to turn your Raspberry PI into a home-assistant enabled secure hub.  I have assumed that you have already installed home-assistant.  You will create your own subdomains with letsencrypt TLS certificates to communicate with your Home Assistant server running on a Raspberry PI, with a Mosquitto MQTT broker and OwnTracks Android app running on your phone.

Goto duckdns and register **one subdomain** for Home Assistant and **another subdomain** for Mosquitto broker and setup cronttabs to auto renew the subdomains and certificates.  Follow this to setup duckdns and to obtain letsencrypt certificates.

https://home-assistant.io/blog/2015/12/13/setup-encryption-using-lets-encrypt/


I've added these two entries into my crontab on the PI to renew the subdomain and certificates.
```
sudo crontab -e
```

```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
40 11,23 * * * ~/certbot/certbot-auto renew --quiet --no-self-upgrade >> ~/certbot/certbot_renew.log
```

# Open up ports on your router
To create port forwards, add entries in your router under NAT Forwarding, Virtual Servers.


Open up port 443 (https) to map to internal port 8123 of the PI(port your Home Assistant is running on). This will allow you to hit **https://homeassistantsubdomain.duckdns.org** to bring up the Home Assistant login screen.


Open up port 80 to map to internal port 80 of the PI allow easy renewal of the subdomain and letsencrypt certificates using crontab jobs.  Port 80 was opened to allow easy renewal of subdomains and certificates. Letsencrypt requires it open when you renew the certificates.  


Open up port 8443 to map to internal port 8883(default port that mosquitto runs on in the Raspberry PI) of the PI to allow clients to communicate with your mosquitto broker using TLS which is running on port 8883 of the Raspberry PI.  Your mosquitto host will be **mosquittosubdomain.duckdns.org** with port 8443 (Note this is your external port that you have opened up in your router)


So now you will have **homeassistantsubdomain.duckdns.org** registered which will point to your Home Assistant on the Raspberry PI.  You will also have another **mosquittosubdomain.duckdns.org** registered which will point to your mosquitto server port on the Raspberry PI.


In your Home Assistant configuration.yaml you should have the ssl certificates setup. These are the default paths when using letsencrypt:
```
http:
  api_password: YOUR_PASSWORD
  ssl_certificate: /etc/letsencrypt/live/homeassistantsubdomain.duckdns.org/fullchain.pem
  ssl_key: /etc/letsencrypt/live/homeassistantsubdomain.duckdns.org/privkey.pem
```

# Setup Mosquitto MQTT broker on the Raspberry PI
Follow http://owntracks.org/booklet/guide/broker/

NB: Only follow the steps until "create a mosquitto user database".  **DO NOT** create certificates using owntracks/tools in their repo. We have already created our letsencrypt certificates.

## Stop mosquitto server
```
sudo service mosquitto stop
```

Change ownership of the mosquittosubdomain certificates to mosquitto user.
```
cd /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/
sudo chown mosquitto *
sudo chgrp mosquitto *
```

## Edit the /etc/mosquitto/conf.d/mosquitto.conf
NB I have commented out tlsv1.  This will mean it will use any tls version. Refer https://mosquitto.org/man/mosquitto-conf-5.html for full config info.

mosquitto.conf
```
listener 8883
#tls_version tlsv1
cafile /etc/ssl/certs/DST_Root_CA_X3.pem
certfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/fullchain.pem
keyfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/privkey.pem
require_certificate true
```

The complete mosquitto.conf is:
```
allow_anonymous false
autosave_interval 1800

connection_messages true
log_type error
log_type warning
log_type notice
log_type information
log_type all
log_type debug
log_timestamp true

persistence true
persistence_file mosquitto.db
persistent_client_expiration 1m

retained_persistence true

listener 8883
#tls_version tlsv1
cafile /etc/ssl/certs/DST_Root_CA_X3.pem
certfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/fullchain.pem
keyfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/privkey.pem
require_certificate true
```

## Start mosquitto server
```
sudo service mosquitto start
```

There should be no errors in the mosquitto logs:
```
/var/log/mosquitto/mosquitto.log
```

## Test connectivity to mosquitto broker
Test mosquitto from the same raspberry PI server that mosquiotto is running on.  This will do an external call via the internet. You can also do this call from another computer, just copy the certs onto there.
```
sudo mosquitto_sub -h mosquittosubdomain.duckdns.org -v -t \$SYS/broker/bytes/\# -p 8443 --cafile /etc/ssl/certs/DST_Root_CA_X3.pem --cert /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/fullchain.pem --key /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/privkey.pem
```
You should get output like:
```
$SYS/broker/bytes/received 21585
$SYS/broker/bytes/sent 21058
$SYS/broker/bytes/received 21649
$SYS/broker/bytes/sent 21133
```

## Setup OwnTracks on your Android phone
Follow the Android section in http://owntracks.org/booklet/features/tlscert/


I used the fullchain.pem and privkey.pem from my /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/ directory.

Generate the users key and certificate using the PKCS#12 container format.
```
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -name "mymosquittocert" -out mymosquittocert.p12
```
NB: Make sure you specify a password for this file. You will need it later when you refer to it in your Android owntracks app.

Copy this file mymosquittocert.p12 to your Android phone. Also copy /etc/ssl/certs/DST_Root_CA_X3.pem from your raspberry PI to your Android device.  You don't need to install these certificates on your phone, just reference them in owntracks preferences.


If you don't have the DST_Root_CA_X3.pem certificate, you can get it from here. https://mosquitto.org/2015/12/using-lets-encrypt-certificates-with-mosquitto/
Look for a pastbin ref in the first comment.

### Install owntracks app.
#### Preferences configuration
```
Preferences->Connection
  ->Mode->Private MQTT
  ->Host-> mosquittosubdomain.duckdns.org
  ->Port-> 8443 (this is the port we openend up on our router for mosquitto)
  No websockets
```
```
Preferences->Connection
  ->Security->TLS
  ->CA Certificate->DST_Root_CA_X3.pem
  ->Client certificate-> mymosquittocert.p12
  ->Client certificate password-> your password from above.
```
Everything else is default.

That should be it. Now you should see connections from your mobile in the mosquitto logs.


When the Android owntracks app connects to the mosquitto server, you will see something like this in the /var/log/mosquitto/mosquitto.log:
```
1500804259: New client connected from xxx.xxx.xxx.xxx as Androidphone (c0, k3600, u'Android').
1500804259: Sending CONNACK to Androidphone (0, 0)
1500804259: Sending PUBLISH to Androidphone (d0, q0, r0, m0, 'owntracks/Android/phone', ... (32 bytes))
1500804259: Received PUBLISH from Androidphone (d1, q1, r1, m81, 'owntracks/Android/phone', ... (126 bytes))
1500804259: Sending PUBACK to Androidphone (Mid: 81)
1500804259: Sending PUBLISH to Androidphone (d0, q1, r0, m81, 'owntracks/Android/phone', ... (126 bytes))
1500804259: Received SUBSCRIBE from Androidphone
1500804259:     owntracks/+/+ (QoS 2)
1500804259: Androidphone 2 owntracks/+/+
1500804259:     owntracks/+/+/info (QoS 2)
1500804259: Androidphone 2 owntracks/+/+/info
1500804259:     owntracks/Android/phone/cmd (QoS 2)
1500804259: Androidphone 2 owntracks/Android/phone/cmd
1500804259:     owntracks/+/+/event (QoS 2)
1500804259: Androidphone 2 owntracks/+/+/event
1500804259:     owntracks/+/+/waypoint (QoS 2)
1500804259: Androidphone 2 owntracks/+/+/waypoint
1500804259: Sending SUBACK to Androidphone
1500804259: Sending PUBLISH to Androidphone (d0, q1, r1, m82, 'owntracks/Android/phone', ... (126 bytes))
1500804259: Received PUBLISH from Androidphone (d0, q1, r1, m83, 'owntracks/Android/phone', ... (134 bytes))
1500804259: Sending PUBACK to Androidphone (Mid: 83)
1500804259: Sending PUBLISH to Androidphone (d0, q1, r0, m83, 'owntracks/Android/phone', ... (134 bytes))
1500804259: Received PUBACK from Androidphone (Mid: 81)
1500804259: Received PUBACK from Androidphone (Mid: 82)
1500804259: Received PUBACK from Androidphone (Mid: 83)
1500804264: Received PUBLISH from Androidphone (d0, q1, r1, m84, 'owntracks/Android/phone', ... (126 bytes))
```

## Setup OwnTracks on your iOS device.
I followed the iOS section in http://owntracks.org/booklet/features/tlscert/
Unfortunately I coudln't get it working.
I installed DST_Root_CA_X3.pem as the "TLS CA certificate", with my p12 file renamed as mymosquittocert.p12.otrp.

The error on my mosquitto log is:
```
1500788145: OpenSSL Error: error:140890B2:SSL routines:SSL3_GET_CLIENT_CERTIFICATE:no certificate returned
```
Please let me know if you know how to get this working.  

## Configure home-assistant to use mosquitto MQTT broker
WIP, not sure if I need to use the generated certificates?
Need to do more reading... Pls let me know if you have any pointers.

## Create an iBeacon Transmitter with the Raspberry Pi
You can also create an iBeacon Transmitter with your PI so that owntracks will detect when you come home, reliably, rather than relying on GPS.

http://www.wadewegner.com/2014/05/create-an-ibeacon-transmitter-with-the-raspberry-pi/

### Author
Nilath Jayamaha
