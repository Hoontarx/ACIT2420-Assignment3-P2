# ACIT2420-Assignment3

## Creating New Droplets
On the digital ocean website:
1. Click **Create** > **Droplets**
2. Click **San Francisco**
3. Select **Datacenter 3 * SFO3** (or whatever region is closest to you)
4. Click **Custom images**
5. Upload and Select **Arch Linux cloudimg** ending with **.qcow2**
6. Select **Premium AMD** > **$7/mo**
7. Select a **SSH Key**
8. Increase Droplet quantity to **2**
9. Enter a **Hostname**
10. Type **web** in the tag textbox
11. Click **Create Droplet**
---
## Creating A Load Balancer
On the digital ocean website:
1. Click **Create**
2. Click **Load Balancers**
3. Select **Regional**
4. Select **San Francisco SFO3**
5. Select **External (Public)**
6. Enter the **web** tag
7. Click **Create Load Balancer**
---
## Creating New System User
Next, we will create the webgen user in the two droplets we just made. This will be a system user that we will use for the purpose of running the nginx server.

You can run the following:
```bash
sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin webgen
```

Now that we have the new user, we are going to add the necessary file structure to their home directory:
```bash
sudo mkdir -p /var/lib/webgen/bin
sudo mkdir -p /var/lib/webgen/HTML
sudo mkdir -p /var/lib/webgen/documents
```

We will now copy the script file into the webgen users home directory in their bin folder:
```bash
sudo cp /path/of/script/file /var/lib/webgen/bin
```

Also, add two test files to the documents folder. These files just need to contain dummy text and make sure the text is different in both files.
```bash
sudo nvim /var/lib/webgen/documents/file-one
sudo nvim /var/lib/webgen/documents/file-two
```

Now that webgens home directory is fully set up, we need to change the ownership to webgen:
```bash
sudo chown -R webgen:webgen /var/lib/webgen
```

This was done as creating a new system user allows for everything to be run with only the necessary privileges, and it gives us a clean working directory to easily manage the files.

## Creating .service & .timer Files
For the second task, we are going to be creating a .service file and a .timer file that will run our script automatically everyday at 5am.

First lets create the generate-index.service file:
```bash
nvim generate-index.service
```

Now inside the .service file we will add the following:
```nvim
[Unit]
Description=Running the generate_index script
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=webgen
Group=webgen
ExecStart=/var/lib/webgen/bin/generate_index

[Install]
WantedBy=multi-user.target
```

Next we will create the generate-index.timer file:
```bash
nvim generate-index.timer
```

Now inside the .timer file we will add the following:
```nvim
[Unit]
Description=Running the generate_index script everyday at 5 am

[Timer]
OnCalendar=*-*-* 5:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Now that we have both the .service and .timer files, we need to put them in the correct directory:
```bash
sudo cp generate-index.service /etc/systemd/system/
```

```bash
sudo cp generate-index.timer /etc/systemd/system/
```

Then we will reload systemd:
```bash
sudo systemctl daemon-reload
```

We will then start and enable the service:
```bash
sudo systemctl start generate-index.service
```

```bash
sudo systemctl enable generate-index.service
```

We will do the same for the .timer file as well:
```bash
sudo systemctl start generate-index.timer
```

```bash
sudo systemctl enable generate-index.timer
```

Lastly, double check the .service and .timer were started correctly with the following:
```bash
sudo systemctl status generate-index.service
```

```bash
sudo systemctl status generate-index.timer
```

If there was an issue, you can use the following to check the logs:
```bash
sudo journalctl -u generate-index.service
```

```bash
sudo journalctl -u generate-index.timer
```
## Setting Up Nginx Config
Now we will be setting up an nginx server that will run using the webgen user.

If you don't have nginx installed you can run:
```bash
sudo pacman -S nginx
```

First, we will be creating a separate server block file that will configure the nginx server. We will need to make a couple of directories:
```bash
sudo mkdir /etc/nginx/sites-available
```

```bash
sudo mkdir /etc/nginx/sites-enabled
```

Now, create the server block configuration file:
```bash
sudo nvim /etc/nginx/sites-available/server-block.conf
```

Add the following to the server-block.conf file:
```nvim
server {
	listen 80;
	listen [::]:80;
	server_name _;
	root /var/lib/webgen/HTML;
	index index.html;

	location / {
		try_files $uri $uri/ =404;
	}

	location /documents {
		alias /var/lib/webgen/documents;
		autoindex on;
		autoindex_exact_size off;
		autoindex_localtime on;
	}
}
```

This server block file is being created as it will allow for different sites to easily be enabled and disabled.

Next, edit the main nginx configuration file to change the user to webgen and have it listen for the new server configuration:
```bash
sudo nvim /etc/nginx/nginx.conf
```

Inside the configuration file, look for `user http;` and change it to:
```nvim
user webgen;
```

Add the following to the http block in the configuration file:
```nvim
http {
	...
	#General Configuration
	charset utf-8;
	types_hash_max_size 4096;
	tcp_nopush on;
	tcp_nodelay on;
	log_not_found off;
	
	#logging access and errors
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log warn;

	#load configurations
	include sites-enabled/*;
}
```

Next, a symbolic link will need to be made:
```bash
sudo ln -s /etc/nginx/sites-available/server-block.conf /etc/nginx/sites-enabled/server-block.conf
```

Before starting the nginx.service file, test the nginx configuration with the following:
```bash
sudo nginx -t
```

If there are no syntax issues, restart daemon and start and enable nginx:
```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl start nginx
```

```bash
sudo systemctl enable nginx
```

Then check the status of nginx with the following:
```bash
sudo systemctl status nginx
```

If you want or need more detail, run the following:
```bash
sudo journalctl -u nginx
```
## Setting Up ufw
Now that the nginx server is fully up and running, we will be setting up a firewall with ufw.

If ufw isn't installed on your system, you can run:
```bash
sudo pacman -S ufw
```

Next, allow SSH and HTTP from anywhere:
```bash
sudo ufw allow ssh
```

```bash
sudo ufw allow http
```

After, enable SSH rate limiting with the following:
```bash
sudo ufw limit ssh
```

Now, enable the firewall with the rules that were just set with:
```bash
sudo ufw enable
```

You can check the status of the firewall with the following:
```bash
sudo ufw status verbose
```