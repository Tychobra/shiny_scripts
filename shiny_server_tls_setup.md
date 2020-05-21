### create the VM

Go to the Google Compute Engine (GCE) page at cloud.google.com and create a new instance:
 
 - choose region us-east-1d (this is our default at Tychobra.  Note: you should try to deploy all
 your GCP services in the same region to reduce latency)
 - select Ubuntu 18.04
 - allow both http and https traffic when configuring the server.
 - just select defaults for everything else

### install software on your new VM

ssh into the instance using the "SSH" button next to the instance in the GCE GUI and

 - install the nano text editor
```
# terminal
sudo apt-get update
sudo apt-get install nano
```

 - install R
 - add the following lines to "/etc/apt/sources.list"
 
```
# in "/etc/apt/sources.list"
deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran35/
deb https://mirror.nodesdirect.com/ubuntu/ bionic-backports main restricted universe
```
 - sign CRAN Ubuntu archives (this allows you to download the latest version of R and the latest R packages)
 
```
# terminal
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9
```

 - install actual R software
 
```
# terminal
sudo apt-get update
sudo apt-get install r-base
```

 - install Shiny (Specific version required to source `R` directories, but not break `shinyWidgets`)
   - Reference: https://github.com/rstudio/shiny/commit/210c248264cb3dab714b65c060486a0462f1cfed
```
# terminal

sudo R
```

```
# R console

install.packages('remotes')
remotes::install_github('rstudio/shiny', ref = '210c248264cb3dab714b65c060486a0462f1cfed')
```


 - **DEPRECATED:** install Shiny from CRAN
```
# terminal
sudo su - \
-c "R -e \"install.packages('shiny', repos='https://cran.rstudio.com/')\""
```

 - install shiny-server open source

```
# terminal

# install gdebi which will allow installation of shiny-server
sudo apt-get install gdebi-core

# download and install the shiny-server software
wget https://download3.rstudio.org/ubuntu-14.04/x86_64/shiny-server-1.5.13.944-amd64.deb
sudo gdebi shiny-server-1.5.13.944-amd64.deb
```

To access your shiny-server from the internet you need to enable a firewall rule to open the defualt shiny server port to the internet.  On GCP, go to "NETWORKING > VPC network > Firewall rules" and create a new rule that allows “tcp:3838” (for the apps)

shiny-server should now be serving apps over "http://<out IP address>:3838".  Try to access it from your web browser.  If you are unable to see a welcome to Shiny page, then something is not set up correctly.

### set up custom Shiny apps on your new shiny-server

 - add a static external IP address using GCP console.  This will make it so that our VM always uses the same IP address. You do this by switching the IP from ephemeral to static in "VPC network > External IP addresses"

 - set up the "/srv/shiny-server" folder to be editable and serve shiny apps. Do this by SSHing into the instance and running
 
```
# terminal
sudo chown shiny:shiny /srv/shiny-server
sudo chmod 0777 /srv/shiny-server
```

  - install a few common R package system dependencies (more system dependencies may be necessary depending on which R packages you use)
  
```
# terminal
sudo apt-get update

# for XML package
sudo apt-get install libxml2-dev

# for curl package
sudo apt-get install libcurl4-openssl-dev

# for git2r package with is a devtools dependency
sudo apt-get install libssl-dev -y

# for V8 (optional depending on if you need V8)
sudo apt-get install libv8-dev

# for Postgres (optional depending on if you need Postgres)
sudo apt-get install libpq-dev
```

### Optional - configure custom domain name by adding the IP address to an A record with your
domain name service provider.  

To apply this new domain name and TLS for https, do the following

 -  install nginx
 
```
# terminal
sudo apt-get update
sudo apt-get install nginx
```

### Setting up TLS
  - obtain CA from letsencrypt
  
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

  - add <your domain name> to nginx config file in
  
```
sudo nano /etc/nginx/sites-available/default
# in "/etc/nginx/sites-available/default" set server_name
server_name <your domain name>; # e.g. apps.tychobra.com
```

  - restart nginx
  
```
# terminal
sudo systemctl restart nginx
```


  - install cert for nginx on your domain from letsencrypt using certbot
  
```
# terminal
sudo certbot --nginx -d <your domain name>
```
  In the terminal, you will be promted with a couple questions.  Respond with:
  1. (A)Agree to the terms of servce
  2. (N)No do not agree to share your email
  3. make no further changes to the server configuration

  - secure shiny-server with a reverse proxy and ssl certificate
  - add the context beginning with “map” to the bottom of the “http” context
  
```
sudo nano /etc/nginx/nginx.conf
in "/etc/nginx/nginx.conf"
http {
  …

  map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;

  }
}
```

  - copy and paste the following over what currently exists in “/etc/nginx/sites-enabled/default”:

```
server {
  listen 80 default_server;
  listen [::]:80 default_server ipv6only=on;

  server_name <your domain name>;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl; # managed by Certbot
  server_name <your domain name>;
  ssl_certificate /etc/letsencrypt/live/<your domain name>/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/<your domain name>/privkey.pem; # managed by Certbot
  #include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  #ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_ciphers AES256+EECDH:AES256+EDH:!aNULL;

  location / {
    proxy_pass http://<your IP address>:3838;
    proxy_redirect http://<your IP address>:3838/ https://$host/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 20d;
  }

}
```
  - replace <your IP address> above with the IP address of your VM, and <your domain name> with your domain name

  - test the new configuration
  
```
# terminal

sudo nginx -t

# if above test comes back OK, reload nginx
sudo systemctl restart nginx
```

Visit the shiny app at your https domain name.  Everything should be working.
