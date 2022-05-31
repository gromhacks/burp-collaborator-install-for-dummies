# Burp Collaborator Install For Dummies

This is just a quick guide on how to install your own burp collaborator server. All thr credit goes to Fabio Pires and the orginal blog post at - https://blog.fabiopires.pt/running-your-instance-of-burp-collaborator-server/.

A few updates:

* You can install certbot-auto via "apt" you do now need to run the orginal wget command.
* This guide utilizes [Notify](https://github.com/projectdiscovery/notify) by https://projectdiscovery.io/ to send notifcations to a Slack instance.

## Setting Up Your Own Burp-Collaborator Server


Buy a domain name that you want to utilize for your OOB Detection (Out of Band). GoDaddy.com is probabaly the simpliest and easiest solution. 

*Note - YOU DO NOT NEED TO SPEND A LOT OF MONEY KEEP IT CHEAP*


1. Create a Digital Ocean Droplet (Or a VPS from your Cloud Provider). The following specs are good enough.

 * 1 GB Memory / 1 Intel vCPU / 25 GB Disk / SFO3 - Ubuntu 20.04 x64
 * Enable SSH on the VPS
 * SSH into the VPS


2. Update your machine

`apt update -y`


3. Install Java

`apt install default-jre`


4. Install Golang - https://go.dev/doc/install and Notify

* You need 1.18+ that is why we installed it directly from https://go.dev/doc/install , you may be able to do this via apt install golang , but it doesn't always work.

*Note - Your Golang Installation ".tar.gx" may be different.*

`wget https://go.dev/dl/go1.18.2.linux-amd64.tar.gz`
`rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz`
`export PATH=$PATH:/usr/local/go/bin`


* Next - Add the Go Path to your .bashrc or .profile file

```
# Golang Path Install
export PATH=$PATH:/usr/local/go/bin
```

* Install Notify

`go install -v github.com/projectdiscovery/notify/cmd/notify@latest`

`cp $HOME/go/bin/notify /usr/bin`


5. Make a direcrtory for all of the collaborator scripts and files

`mkdir -p /usr/local/collaborator/`


6. Download the Burpsuite Pro Jar File from Burpsuite

* https://portswigger.net/burp/releases/professional-community-2022-3-9
* SCP the file to the Remote Server

`scp burpsuite_pro_v2022.3.9.jar root@<YOUR OOB SERVER IP>:/usr/local/collaborator/`

*Note - If you do not want to pay for Burp Suite Pro we recommend that you setup your own [interactsh](https://github.com/projectdiscovery/interactsh) instance instead*


7. Create the Config File

* Save it to - /usr/local/collaborator/

`nano /usr/local/collaborator/collaborator.config`

```
{
  "serverDomain" : "subdomain.domain.com", # CHANGE THIS
  "workerThreads" : 10,
  "eventCapture": {
      "localAddress" : [ "XXX.XXX.XXX.XXX" ], # CHANGE THIS (PUBLIC IP OF VPS)
      "publicAddress" : "XXX.XXX.XXX.XXX", # CHANGE THIS (PUBLIC IP OF VPS)
      "http": {
         "ports" : 80
       },
      "https": {
          "ports" : 443
      },
      "smtp": {
          "ports" : [25, 587]
      },
      "smtps": {
          "ports" : 465
      },
      "ssl": {
          "certificateFiles" : [
              "/usr/local/collaborator/keys/privkey.pem",
              "/usr/local/collaborator/keys/cert.pem",
              "/usr/local/collaborator/keys/fullchain.pem" ]
      }
  },
  "polling" : {
      "localAddress" :  "XXX.XXX.XXX.XXX", # CHANGE THIS (PUBLIC IP OF VPS)
      "publicAddress" :  "XXX.XXX.XXX.XXX", # CHANGE THIS (PUBLIC IP OF VPS)
      "http": {
          "port" : 9090
      },
      "https": {
          "port" : 9443
      },
      "ssl": {
          "certificateFiles" : [
              "/usr/local/collaborator/keys/privkey.pem",
              "/usr/local/collaborator/keys/cert.pem",
              "/usr/local/collaborator/keys/fullchain.pem" ]

      }
  },
  "dns": {
      "interfaces" : [{
          "name":"ns1.subdomain.domain.com", # CHANGE THIS
          "localAddress":"XXX.XXX.XXX.XXX", # CHANGE THIS (PUBLIC IP OF VPS)
          "publicAddress":"XXX.XXX.XXX.XXX" # CHANGE THIS (PUBLIC IP OF VPS)
      }],
      "ports" : 53
   },
   "logLevel" : "DEBUG"
}
```


8. Create Certificates and Configure_certs.sh script

`cd /usr/local/collaborator/`
`apt install certbot`
`nano /usr/local/collaborator/configure_certs.sh`

* Then copy and past the following into the configure_certs.sh script

```
#!/bin/bash

CERTBOT_DOMAIN=$1
if [ -z $1 ];
then
    echo "Missing mandatory argument. "
    echo " - Usage: $0  <domain> "
    exit 1
fi
CERT_PATH=/etc/letsencrypt/live/$CERTBOT_DOMAIN/
mkdir -p /usr/local/collaborator/keys/

if [[ -f $CERT_PATH/privkey.pem && -f $CERT_PATH/fullchain.pem && -f $CERT_PATH/cert.pem ]]; then
        cp $CERT_PATH/privkey.pem /usr/local/collaborator/keys/
        cp $CERT_PATH/fullchain.pem /usr/local/collaborator/keys/
        cp $CERT_PATH/cert.pem /usr/local/collaborator/keys/
        chown -R collaborator /usr/local/collaborator/keys
        echo "Certificates installed successfully"
else
        echo "Unable to find certificates in $CERT_PATH"
fi
```


9. Run Certbot


*Note - do not skip to Step 11, otherwise your DNS TXT records may have issues resolving.*

* Enter your email and regional information when prompted.
* Next follow the instructions. You will be promopted to add TXT records to your domain provider. Ensure you give these records about 5 minutes to populate before proceeding.

`certbot certonly -d subdomain.domain.com -d *.subdomain.domain.com  --server https://acme-v02.api.letsencrypt.org/directory --manual --agree-tos --no-eff-email --manual-public-ip-logging-ok --preferred-challenges dns-01`



10. Configure and Install Certs

`chmod +x /usr/local/collaborator/configure_certs.sh && /usr/local/collaborator/configure_certs.sh subdomain.domain.com`


11. Update your DNS Records to point torward your Collaborator Server VPS

* An "NS" record point to ns1.subdomain.domain.com (Example: subdomain.domain.com --> ns1.subdomain.domain.com)
* An "A" record point to the Collaborator Server Public IP (Example: ns1.subdomain.domain.com --> xxx.xxx.xxx.xxx)


12. Start/ Test the server

*Note - Your BurpSuite Version may be different*

`/usr/bin/java -Xms10m -Xmx200m -XX:GCTimeRatio=19 -jar /usr/local/collaborator/burpsuite_pro_v2022.3.9.jar --collaborator-server --collaborator-config=/usr/local/collaborator/collaborator.config`

* Once complete exit and kill the process.

*Note - sometimes port 53 is already taken by another service by default on Ubuntu, if that is the case you can follow the instructions in this [blog post](https://www.linuxuprising.com/2020/07/ubuntu-how-to-free-up-port-53-used-by.html). You want to do this last otherwise you will not be able to reach the internet. Keep this in mind if you do future updates*


13. Run a Health Check via your Burpsuite Client (Local)

* Your Local Burp Suite Client > Project options > Misc > Burp Collaborator Server > Use a private collaborator server > Update the Following:
  
    Server Location - subdomain.domain.com 
    Polling Location - subdomain.domain.com:9443


14. Create your own Slack Channel/Instance (Free Tier is Good Enough). 

* Create a App with a Webhook - https://api.slack.com/apps?new_app=1
* Once complete put all contents into the $HOME/.config/notify/provider-config.yaml file for "Notify".

```
slack:
  - id: "slack"
    slack_channel: "CHANNEL NAME"
    slack_username: "BOT USERNAME"
    slack_format: "{{data}}"
    slack_webhook_url: "https://hooks.slack.com/services/XXXXXX"

```


15. Create a bash script that can be run Notify and the Collaborator Instance via "systemctl".

`nano /usr/local/collaborator/collaborator_start`

```
#!/bin/bash

/usr/bin/java -Xms10m -Xmx200m -XX:GCTimeRatio=19 -jar /usr/local/collaborator/burpsuite_pro_v2022.3.9.jar --collaborator-server --collaborator-config=/usr/local/collaborator/collaborator.config | /usr/bin/notify -silent

```

`chmod +x /usr/local/collaborator/collaborator_start`

*Note - we run notify in "silent" mode in order to prevent the banner from showing*


16. Create a "systemctl" script

`nano /etc/systemd/system/collaborator.service`

```
[Unit]
Description=Burp Collaborator Server Daemon
After=network.target

[Service]
Type=simple
User=root
UMask=777
ExecStart=/usr/bin/java -Xms10m -Xmx200m -XX:GCTimeRatio=19 -jar /usr/local/collaborator/burpsuite_pro_v2022.3.9.jar --collaborator-server --collaborator-config=/usr/local/collaborator/collaborator.config
Restart=on-failure

# Configures the time to wait before service is stopped forcefully.
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

`systemctl enable collaborator`
`systemctl start collaborator`


17. Start/ Test the server and run another Health Check via your Burpsuite Client (Local). This should trigger Slack Notifications.

18. Configure firewall rules to only allowlist your IPs etc.

Happy Hacking!!
