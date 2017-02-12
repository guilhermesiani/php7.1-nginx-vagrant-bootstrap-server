# Provision a Nginx server with PHP7.1 and MySQL^5
```
$ mkdir DIRECTORY
$ cd DIRECTORY
$ vagrant init
$ vagrant box add ubuntu/trusty64
```
In the same directory create ur Vagrantfile with this content:
```
Vagrant.configure(2) do |config|
  	config.vm.box = "ubuntu/trusty64"
	config.vm.provision :shell, path: "bootstrap.sh"
	config.vm.network :forwarded_port, guest: 80, host: 8080
	config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"
end
```
And bootstrap.sh file with the content:
```
#!/usr/bin/env bash

apt-get update
if ! [ -L /var/www ]; then
    rm -rf /var/www
    ln -fs /vagrant /var/www
fi

# Install nginx
apt-get install -y nginx

# Install mysql
debconf-set-selections <<< "mysql-server mysql-server/root_password password root"
debconf-set-selections <<< "mysql-server mysql-server/root_password_again password root"
apt-get install -y mysql-server

# Install php-fpm
add-apt-repository -y ppa:ondrej/php && sudo apt-get update
apt-get install -y php7.1-cli php7.1-fpm php7.1-mysql php7.1-curl php-memcached php7.1-dev php7.1-mcrypt php7.1-sqlite3 php7.1-mbstring
apt-cache search php7.1

# php.ini
sed -i.bak 's/^;cgi.fix_pathinfo.*$/cgi.fix_pathinfo = 1/g' /etc/php/7.1/fpm/php.ini

service php7.1-fpm restart

# Configure host
cat << 'EOF' > /etc/nginx/sites-available/default
server {
	# Port that the web server will listen on.
	listen 80;

	# Host that will serve this project.
	server_name localhost;

	# Useful logs for debug.
	access_log /var/log/nginx/localhost_access.log;
	error_log /var/log/nginx/localhost_error.log;
	rewrite_log on;

	# The location of our projects public directory.
	root /usr/share/nginx/html;

	# Point index to the Laravel front controller.
	index index.php index.html;

	location / {
		# URLs to attempt, including pretty ones.
		try_files $uri $uri/ /index.php?$query_string;
	}

	# Remove trailing slash to please routing system.
	if (!-d $request_filename) {
		rewrite ^/(.+)/$ /$1 permanent;
	}

	# PHP FPM configuration.
	location ~* \.php$ {
		fastcgi_pass unix:/run/php/php7.1-fpm.sock;
		fastcgi_index index.php;
		fastcgi_split_path_info ^(.+\.php)(.*)$;
		include /etc/nginx/fastcgi_params;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	}

	# We don't need .ht files with nginx.
	location ~ /\.ht {
		deny all;
	}

	# Set header expirations on per-project basis
	location ~* \.(?:ico|css|js|jpe?g|JPG|png|svg|woff)$ {
		expires 365d;
	}
}
EOF

# Restart servers
service nginx restart
service php7.1-fpm restart
```
Last one command: ```vagrant up```

After command ```vagrant up``` or ```vagrant provision``` use this url to access your php7 app from vagrant: http://localhost:8080 or http://127.0.0.1:8080

Success!!
Now you are seeing the Nginx welcome page
