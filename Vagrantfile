# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
	# base box
	config.vm.box = "debian/jessie64"

	# forward localhost:8080 to guest port 80
	config.vm.network "forwarded_port", guest: 80, host_ip: "127.0.0.1", host: 8080

	# virtualbox configuration
	config.vm.provider "virtualbox" do |vb|
 		# Display the VirtualBox GUI when booting the machine
		# vb.gui = true
    
		# Customize name of the VM:
		vb.name = "dokuwiki"

		# Use linked clone instead of full copy
		vb.linked_clone = true
    
		# Customize the amount of memory on the VM:
		vb.memory = "256"

		# Customize number of CPUs
		vb.cpus = 1

		# Set execution cap at 50%
		vb.customize ["modifyvm", :id, "--cpuexecutioncap", "30"]
	end

	# configure system
	config.vm.provision "shell", inline: <<-SHELL
		export DEBIAN_FRONTEND=noninteractive

		echo "===> Update packages"
		sudo apt-get --quiet              update

		echo "===> Install system packages"
		sudo apt-get --quiet --assume-yes install nginx 
		sudo apt-get --quiet --assume-yes install uwsgi uwsgi-plugin-php
		sudo apt-get --quiet --assume-yes install ufw

		echo "===> Configure firewall"
		sudo ufw allow 80/tcp
		sudo ufw allow 22/tcp
		sudo ufw limit 22/tcp
		sudo ufw default deny  incoming
		sudo ufw default allow outgoing
		sudo ufw --force enable

		echo "===> Configure additional PHP settings"
		sudo sed -i -e "/upload_max_filesize/ {s/\\(.*\\) = .*/\\1 = 25M/}" /etc/php5/embed/php.ini
		sudo sed -i -e "/post_max_size/       {s/\\(.*\\) = .*/\\1 = 30M/}" /etc/php5/embed/php.ini

		echo "===> Install and configure Dokukiwi"
		sudo mkdir -p /srv/dokuwiki
		sudo cp -a /vagrant/dokuwiki/. /srv/dokuwiki
		# sudo curl -s -o /srv/dokuwiki-stable.tgz http://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
		# sudo tar --extract --gzip --file /srv/dokuwiki-stable.tgz --strip-components 1 --directory /srv/dokuwiki

		cat <<-EOF | sudo tee /srv/dokuwiki/conf/local.php
			<?php
			\\$conf['userewrite'] = 1;
			?>
		EOF

		sudo chown -R www-data:www-data /srv/dokuwiki

		echo "===> Configure uWSGI"
		cat <<-EOF | sudo tee /etc/uwsgi/apps-available/php.ini
			[uwsgi]
			plugins = php
			chown-socket = www-data:www-data
			processes = 4
		EOF

		sudo ln -s /etc/uwsgi/apps-available/php.ini /etc/uwsgi/apps-enabled/

		sudo systemctl restart uwsgi

		echo "===> Configure nginx"
		sudo unlink /etc/nginx/sites-enabled/default
     
		cat <<-EOF | sudo tee /etc/nginx/sites-available/dokuwiki
		server {
		  listen 80 default_server;

		  server_name dokuwiki.local;

		  client_max_body_size 30M;

		root /srv/dokuwiki;
		  index doku.php;

		  location / {
		    try_files \\$uri @dokuwiki;
		  }

		  location @dokuwiki { 
		    rewrite ^/_media/(.*)          /lib/exe/fetch.php?media=\\$1 last;
		    rewrite ^/_detail/(.*)         /lib/exe/detail.php?media=\\$1 last;
		    rewrite ^/_export/([^/]+)/(.*) /doku.php?do=export_\\$1&id=\\$2 last;
		    rewrite ^/(?!lib/)(.*)         /doku.php?id=\\$1&\\$args last;
		  } 

		  location ~ /(COPYING|README|VERSION|install.php) { 
		    deny all; 
		  }

		  location ~ /(conf|bin|inc)/ {
		    deny all;
		  }
    
		  location ~ /data/ {
		    internal;
		  }

		  location ~ \\.php\\$ {
		    if (!-f \\$request_filename) { return 404; }

		    include uwsgi_params;
		    uwsgi_modifier1 14;
		    uwsgi_pass unix:/run/uwsgi/app/php/socket;
		  }
		}
		EOF

		sudo ln -s /etc/nginx/sites-available/dokuwiki /etc/nginx/sites-enabled/
		sudo systemctl reload nginx
	SHELL
end