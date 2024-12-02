# ACIT2420-Assignment3

## Task 1 - Creating New System User
We are going to start by creating a new system user so we can store and run the generate_index script file inside their home directory in their bin folder.

First we are going to create the new system user using useradd, specifying their home directory and an appropriate shell for the new system user.

You can run the following:
```bash
sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin
```

Now that we have the new user, we are going to add the necessary file structure to their home directory:
```bash
sudo mkdir -p /var/lib/webgen/bin
sudo mkdir -p /var/lib/webgen/HTML
```

We will now copy the script file into the webgen users home directory in their bin folder:
```bash
sudo cp /path/of/script/file /var/lib/webgen/bin
```

Now that webgens home directory is fully set up, we need to change the ownership to webgen:
```bash
sudo chown -R webgen:webgen /var/lib/webgen
```

This was done as creating a new system user allows for everything to be run with only the necessary privileges, and it gives us a clean working directory to easily manage the files.

## Task 2 - Creating .service & .timer Files
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
## Task 3 - Setting Up nginx Config
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
## Task 4 - Setting Up ufw
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

# Notes:
When adding the user for additional system information, I used whoami assuming it would just show the current user, but I guess since the script is owned by webgen it shows webgen as the user? or maybe it's because the nginx server is set to webgen?