# Setup subdomains and letsencrypt certificates
This will let you setup Letsencrypt TLS certificates to communicate with your Home Assistant server running on the Raspberry PI, and a Mosquitto MQTT broker and OwnTracks andriod app running on your phone.

Goto duckdns and register one subdomain for Home Assistant and another subdomain for Mosquitto broker and setup cronttabs to auto renew the subdomains and certificates.  Follow this to setup duckdns and to obtain letsencrypt certificates.

https://home-assistant.io/blog/2015/12/13/setup-encryption-using-lets-encrypt/

# Open up ports on your router
Open up 443 (https) to map to internal port 8123 of the PI(port your Home Assistant is running on). This will allow you to hit https://homeassistantsubdomain.duckdns.org to bring up the Home Assistant login screen.
Open up 80 to map to internal port 80 of the PI allow easy renewal of the subdomain and letsencrypt certificates using crontab jobs.
Open up 8443 to map to internal port 8883(default port that mosquitto runs on in the Raspberry PI) of the PI to allow clients to communicate with your mosquitto broker using TLS which is running on port 8883 of the Raspberry PI.  Your mosquitto host will be mosquittosubdomain.duckdns.org with port 8443 (Note this is your external port that you have opened up in your router)

Open up 80 to allow easy renewal of subdomains and certificates.  

So now you will have homeassistantsubdomain.duckdns.org registered which will point to your Home Assistant on the Raspberry PI.  You will also have another mosquittosubdomain.duckdns.org registered which will point to your mosquitto server port on the Raspberry PI.

In your Home Assistant configuration.yaml you should have(default paths):
http:
  api_password: YOUR_PASSWORD
  ssl_certificate: /etc/letsencrypt/live/homeassistantsubdomain.duckdns.org/fullchain.pem
  ssl_key: /etc/letsencrypt/live/homeassistantsubdomain.duckdns.org/privkey.pem


# Setup Mosquitto MQTT broker on the Raspberry PI
Follow http://owntracks.org/booklet/guide/broker/

NB: Only follow the steps until "create a mosquitto user database".  DO NOT create certificates using owntracks/tools in their repo. We have already created our letsencrypt certificates.

## Stop mosquitto server
sudo service mosquitto stop

Change ownership of the mosquittosubdomain certificates to mosquitto user.
cd /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/
sudo chown mosquitto *
sudo chgrp mosquitto *

## Edit the mosquitto.conf
NB I have commented out tlsv1.  This will mean it will use any tls version. Refer https://mosquitto.org/man/mosquitto-conf-5.html for full config info.

listener 8883
#tls_version tlsv1
cafile /etc/ssl/certs/DST_Root_CA_X3.pem
certfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/fullchain.pem
keyfile /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/privkey.pem
require_certificate true

## Start mosquitto server
sudo service mosquitto start

There should be no errors in the mosquitto logs:
/var/log/mosquitto/mosquitto.log

## Test connectivity to mosquitto broker
Test mosquitto from the same raspberry PI server that mosquiotto is running on.  This will do an external call via the internet. You can also do this call from another computer, just copy the certs onto there.
sudo mosquitto_sub -h mosquittosubdomain.duckdns.org -v -t \$SYS/broker/bytes/\# -p 8443 --cafile /etc/ssl/certs/DST_Root_CA_X3.pem --cert /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/fullchain.pem --key /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/privkey.pem

You should get output like:
$SYS/broker/bytes/received 21585
$SYS/broker/bytes/sent 21058
$SYS/broker/bytes/received 21649
$SYS/broker/bytes/sent 21133

## Setup OwnTracks on your Andriod phone
Follow the Andriod section in http://owntracks.org/booklet/features/tlscert/
I used the fullchain.pem and privkey.pem from my /etc/letsencrypt/live/mosquittosubdomain.duckdns.org/ directory.

openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -name "mymosquittocert" -out mymosquittocert.p12
NB: Make sure you specify a password for this file. You will need it later when you refer to it in your andriod owntracks app.

Copy this file mymosquittocert.p12 to your andriod phone. Also copy /etc/ssl/certs/DST_Root_CA_X3.pem from your raspberry PI to your Andriod device.  
If you don't have the DST_Root_CA_X3.pem certificate, you can get it from here. https://mosquitto.org/2015/12/using-lets-encrypt-certificates-with-mosquitto/
Look for a pastbin ref in the first comment.

### Install owntracks app.
Preferences->Connection
  ->Mode->Private MQTT
  ->Host-> mosquittosubdomain.duckdns.org
  ->Port-> 8443 (this is the port we openend up on our router for mosquitto)
  No websockets

Preferences->Connection
  ->Security->TLS
  ->CA Certificate->DST_Root_CA_X3.pem
  ->Client certificate-> mymosquittocert.p12
  ->Client certificate password-> your password from above.

Everything else is default.

That should be it. Now you should see connections from your mobile in the mosquitto logs.


## Setup OwnTracks on your iOS device.
I followed the iOS section in http://owntracks.org/booklet/features/tlscert/
Unfortunately I coudln't get this working. The error on my mosquitto log is:
1500788145: OpenSSL Error: error:140890B2:SSL routines:SSL3_GET_CLIENT_CERTIFICATE:no certificate returned
Please let me know if you know how to get this working.  