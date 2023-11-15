1. **Set IP Address and DNS.**
    * `sudo vim /etc/dhcpcd.conf`
      ```
      interface eth0
      static ip_address=<ip address CIDR notation e.g 192.168.1.50/24>
      static routers=<ip address of router>
      static domain_name_servers=1.1.1.1 1.0.0.1
      ```
1. **Install xrdp - Remote Desktop Service.**
    * `sudo apt update`
    * `sudo apt install -y xrdp --fix-missing`
    * `sudo reboot`
1. **Cloudflare installation.**
    * `sudo useradd -s /usr/sbin/nologin -r -M cloudflared`
    * `wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz`
    * `tar -xzvf cloudflared-stable-linux-arm.tgz`
    * `sudo cp ./cloudflared /usr/local/bin`
    * `sudo chmod +x /usr/local/bin/cloudflared`
    * `sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared`
    * `cloudflared -v`
    * `sudo nano /etc/default/cloudflared`
      ```
      # Commandline args for cloudflared
      CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
      ```
    * `sudo nano /lib/systemd/system/cloudflared.service`
      ```
      [Unit]
      Description=cloudflared DNS over HTTPS proxy
      After=syslog.target network-online.target

      [Service]
      Type=simple
      User=cloudflared
      EnvironmentFile=/etc/default/cloudflared
      ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
      Restart=on-failure
      RestartSec=10
      KillMode=process

      [Install]
      WantedBy=multi-user.target
      ```
    * `sudo systemctl daemon-reload`
    * `sudo systemctl enable cloudflared`
    * `sudo systemctl start cloudflared`
    * `sudo systemctl status cloudflared`
    * `nslookup -port=5053 example.com 127.0.0.1`
1. **Install Pi-Hole**
    * `curl -sSL https://install.pi-hole.net | PIHOLE_SKIP_OS_CHECK=true sudo -E bash`
      ```
       │ Configure your devices to use the Pi-hole as their DNS server      │
       │ using:                                                             │
       │                                                                    │
       │ IPv4:        X.X.X.X                                               │
       │ IPv6:        Not Configured                                        │
       │                                                                    │
       │ If you set a new IP address, you should restart the Pi.            │
       │                                                                    │
       │ The install log is in /etc/pihole.                                 │
       │                                                                    │
       │ View the web interface at http://pi.hole/admin or                  │
       │ http://X.X.X.X/admin                                               │
       │                                                                    │
       │ Your Admin Webpage login password is ##########
      ```
1. **Set Pi-Hole to use port 8093 instead of 80.**
    * `sudo cp /etc/lighttpd/lighttpd.conf /etc/lighttpd/lighttpd.conf.backup`
    * `sudo sed -ie "s/= 80/= 8093/g" /etc/lighttpd/lighttpd.conf`
    * `sudo /etc/init.d/lighttpd restart`
1. **Install and configure the latest ddclient**
    * `wget $(curl -s https://api.github.com/repos/ddclient/ddclient/releases/latest | grep 'tarball_' | cut -d\" -f4)`
    * `tar -xvzf $(curl -s https://api.github.com/repos/ddclient/ddclient/releases/latest | grep -m 1 'name"' | cut -d\" -f4)`
    * `sudo cp -f $(ls | grep 'ddclient')/ddclient /usr/sbin/`
    * `sudo mkdir /etc/ddclient/`
    * `sudo nano /etc/ddclient/ddclient.conf`
      ```
      daemon=1800
      syslog=yes
      protocol=cloudflare
      use=web, web=ipinfo.io/ip
      zone=<domain.com>
      ssl=yes
      login=<account e-mail address>
      password='<password>'
      <domain.com>,*.<domain.com>
      ```
    * `sudo service ddclient restart`
    * `sudo ddclient -daemon=0 -debug -verbose -noquiet`
1. **Install snapd**
    * `sudo apt-get -y install snapd`
    * `sudo snap install core; sudo snap refresh core`
    * `sudo apt-get -y remove certbot`
    * `sudo snap install --classic certbot`
    * `sudo ln -s /snap/bin/certbot /usr/bin/certbot`
    * `sudo snap set certbot trust-plugin-with-root=ok`
    * `sudo snap install certbot-dns-cloudflare`
    * `mkdir -p ~/.secrets/certbot/`
    * `nano ~/.secrets/certbot/cloudflare.ini`
      ```
       # Cloudflare API token used by Certbot
       dns_cloudflare_api_token = <api token>
      ```
    * `chmod 600  ~/.secrets/certbot/cloudflare.ini`
    * `sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini -d *.<domain.com>`
    * `sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096`
1. **Install nginx reverse web proxy - used to provide https redirect**
    * `sudo apt-get -y install nginx`
    * `sudo nano /etc/nginx/sites-enabled/<domain.com>.conf`
        ```
        # http -> https
        server {
        	listen 80;
        	server_name *.<domain.com> <domain.com>;
        	return 301 https://$host$request_uri;
        }

        # foundry
        server {
        	listen 443 ssl;
        	server_name daxial.<domain.com>;
        	ssl_certificate /etc/letsencrypt/live/<domain.com>/fullchain.pem;
        	ssl_certificate_key /etc/letsencrypt/live/<domain.com>/privkey.pem;

        	# intermediate configuration
        	ssl_protocols TLSv1.2 TLSv1.3;
        	ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:E       CDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        	ssl_prefer_server_ciphers off;

        	ssl_dhparam /etc/ssl/certs/dhparam.pem;
        	ssl_session_timeout 1d;
        	ssl_session_cache shared:SSL:50m;
        	ssl_stapling on;
        	ssl_stapling_verify on;
        	add_header Strict-Transport-Security max-age=15768000;

        	default_type  application/octet-stream;
        	location / {
        		# pass everything from foundry subdomain to foundry
        		proxy_pass http://127.0.0.1:30000;

        		# gotta do some stuff for the websockets foundry uses
        		proxy_set_header Upgrade $http_upgrade;
        		proxy_set_header Connection "Upgrade";

        		# set proxy headers
        		proxy_set_header Host $host;
        		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        		proxy_set_header X-Forwarded-Proto $scheme;
        	}
        }

        # webserver
        server {
        	listen 443 ssl;
        	server_name *.<domain.com> www.<domain.com> <domain.com>;
        	ssl_certificate /etc/letsencrypt/live/<domain.com>/fullchain.pem;
        	ssl_certificate_key /etc/letsencrypt/live/<domain.com>/privkey.pem;

        	# intermediate configuration
        	ssl_protocols TLSv1.2 TLSv1.3;
        	ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:E       CDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        	ssl_prefer_server_ciphers off;

        	ssl_dhparam /etc/ssl/certs/dhparam.pem;
        	ssl_session_timeout 1d;
        	ssl_session_cache shared:SSL:50m;
        	ssl_stapling on;
        	ssl_stapling_verify on;
        	add_header Strict-Transport-Security max-age=15768000;

        	default_type application/octet-stream;

        	root /var/www/<domain.com>;

        	index index.html index.htm index.nginx-debian.html;

        	location / {
        		try_files $uri $uri/ =404;
        	}

        	# certbot acme
        	location ~ /.well-known {
        		allow all;
        	}
        }
        ```
    * `sudo cat /etc/letsencrypt/live/<domain.com>/privkey.pem \
           /etc/letsencrypt/live/<domain.com>/cert.pem | \
           sudo tee /etc/letsencrypt/live/<domain.com>/combined.pem`
    * `sudo cat /etc/letsencrypt/live/<domain.com>/privkey.pem /etc/letsencrypt/live/<domain.com>/cert.pem | \
           sudo tee /etc/letsencrypt/live/<domain.com>/combined.pem`
    * `sudo chown www-data -R /etc/letsencrypt/live`
1. **Cron job to Cretificate renewal. Scheduled for 1st of month, 0 minutes, 3rd hour AM.**
    * `sudo crontab -e`
    * `0 3 1 * * /usr/bin/certbot renew --quiet`
        ```
        /etc/letsencrypt/renewal-hooks/pre/stop-service.share
        #!/bin/bash
        /bin/systemctl stop nginx
        /bin/systemctl stop ligttpd

        /etc/letsencrypt/renewal-hooks/pre/start-service.share
        #!/bin/bash
        /bin/systemctl start nginx
        /bin/systemctl start ligttpd
        ```
1. **Update Pihole.**
    * `sudo pihole -up`
#### References:
1. [Configure Pi-Hole DNS + Cloudflare DNS over HTTPS (DoH) on a Raspberry Pi](https://nathancatania.com/posts/pihole-dns-doh/)
1. [How to set up DNS-O-MATIC for Cloudflare (and the other way around) and a FritzBox](https://support.opendns.com/hc/en-us/community/posts/115000937008-How-to-set-up-DNS-O-MATIC-for-Cloudflare-and-the-other-way-around-and-a-FritzBox)