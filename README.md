# How to Install ERPNext Open Source ERP on Ubuntu 18.04 on Google Cloud Platform

Refer to https://www.vultr.com/docs/how-to-install-erpnext-open-source-erp-on-ubuntu-17-04-22771 and modify on June, 20, 2018 by Yod (yod@facgure.com). This article based on Ubuntu 18.04 LTS (Bionic Beaver) on Google Cloud Platform.

ERP or Enterprise Resource Planning is an enterprise application suite used to manage core business processes. ERPNext is a free and open source, self-hosted ERP application written in Python with Frappe framework (http://frappe.io/) . It uses Node.js for the front end and MariaDB to store its data. ERPNext provides an easy-to-use web interface that allows businesses to manage day to day tasks. It contains modules for accounting, CRM, HRM, manufacturing, POS, project management, purchasing, sales management, warehouse management, healthcare, and more. ERPNext can be used to manage different industries such as service providers, manufacturing, retail and schools.

# Prerequisites

- An Ubuntu 18.04 LTS (Bionic Beaver) server instance. 
- A sudo user.

Note: For this tutorial, we will use erp.site.io as the domain name pointed to the server. Please make sure to replace all occurrences of erp.site.io with your actual domain name.

Before we begin, make sure your server is up-to-date.

    sudo apt update
    sudo apt -y upgrade

# Install Development Tools

ERPNext needs Python version 2.7 to work. Install Python 2.7.

    sudo apt -y install python-minimal

You should be able to verify its version.

    python -V

You will see the following output.

    Yod@erp.site.io:~$ python -V
    Python 2.7.15rc1

# Install a few more dependencies

    sudo apt -y install git build-essential software-properties-common python-setuptools python-dev libffi-dev libssl-dev vim

# Install Python's pip tool

Pip is the dependency manager for Python packages.

    wget https://bootstrap.pypa.io/get-pip.py
    sudo python get-pip.py

Ensure that you have the latest version of pip and setuptools.

    sudo pip install --upgrade pip setuptools

# Install Ansible using `pip`

Ansible automates software provisioning, configuration management, and application deployment.

    sudo pip install ansible

# Install MariaDB Server

Add the MariaDB repository into the system.

    sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8

    sudo add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://mirror.nodesdirect.com/mariadb/repo/10.2/ubuntu xenial main'

# Install MariaDB.

    sudo apt update
    sudo apt -y install mariadb-server libmysqlclient-dev

Provide a strong password for the MariaDB root user when asked.

The Barracuda storage engine is required for the creation of ERPNext databases, so you will need to configure MariaDB to use the Barracuda storage engine. Edit the default MariaDB configuration file `my.cnf`.

    sudo vi /etc/mysql/my.cnf

Add the following lines under the `[mysqld]` line.

    innodb-file-format=barracuda
    innodb-file-per-table=1
    innodb-large-prefix=1
    character-set-client-handshake = FALSE
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci

Restart MariaDB and enable it to automatically start at boot time.

    sudo systemctl restart mariadb
    sudo systemctl enable mariadb

Before configuring the database, you will need to secure MariaDB. You can secure it by running the `mysql_secure_installation` script.

    sudo mysql_secure_installation

You will be asked for the current MariaDB root password. Provide the password you have set during the installation. You will be asked if you wish to change the existing password of the root user of your MariaDB server. You can skip setting a new password, as you have already provided a strong password during installation. Answer `Y` to all of the other questions that are asked.

Login to MariaDB using root user via `mysql -uroot -p` to ensure that setting is properly work. You will get response like this.

	Enter password:
	Welcome to the MariaDB monitor.  Commands end with ; or \g.
	Your MariaDB connection id is 3
	Server version: 10.1.29-MariaDB-6 Ubuntu 18.04

	Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

	MariaDB [(none)]>
	
For unexpected response like `ERROR 1698 (28000): Access denied for user 'root'@'localhost'` even you ware enter the right password is cause from default authentication plugin for user root, solution is clearly explain in this [link: Authentication Plugin](https://mariadb.com/kb/en/library/authentication-plugin-unix-socket/)

# Install Nginx, Node.js and Redis

Add the Nodesource repository for Node.js 8.x.

    sudo curl --silent --location https://deb.nodesource.com/setup_8.x | sudo bash -

Install Nginx, Node.js and Redis.

    sudo apt -y install nginx nodejs redis-server

Start Nginx and enable it to start at boot time.

    sudo systemctl start nginx
    sudo systemctl enable nginx

Start Redis and enable it to start at boot time.

    sudo systemctl start redis-server
    sudo systemctl enable redis-server

# Install PDF Converter

The `wkhtmltopdf` program is a command line tool that converts HTML into PDF using the QT Webkit rendering engine. Install the required dependencies.

    sudo apt -y install libxrender1 libxext6 xfonts-75dpi xfonts-base

Download the latest version of wkhtmltopdf.

    wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.4/wkhtmltox-0.12.4_linux-generic-amd64.tar.xz
Extract the archive.

    sudo tar -xf wkhtmltox-0.12.4_linux-generic-amd64.tar.xz -C /opt

The above command will extract the archive to `/opt/wkhtmltox`. Create a softlink so that wkhtmltopdf and wkhtmltoimage can be executed globally as a command.

    sudo ln -s /opt/wkhtmltox/bin/wkhtmltopdf /usr/bin/wkhtmltopdf
    sudo ln -s /opt/wkhtmltox/bin/wkhtmltoimage /usr/bin/wkhtmltoimage

You can now run wkhtmltopdf -V to check if it is working, you will see this.

    Yod@erp-site-io:~$ wkhtmltopdf -V
    wkhtmltopdf 0.12.4 (with patched qt)

At this point, we have all the required dependencies installed. You can now proceed to install Bench.

# Install Bench

Bench is a command line utility provided by Frappe to install and manage the ERPNext application on a Unix-based system for both development and production purposes. Bench can also create and manage Nginx and supervisor configurations.

Create a new user to run Bench processes in the isolated environment.

    sudo adduser bench --home /opt/bench

Provide sudo permissions to the bench user.

    sudo usermod -aG sudo bench

Login as the newly created bench user.

    sudo su - bench

Clone the Bench repository in /opt/bench.

    cd /opt/bench
    git clone https://github.com/frappe/bench bench-repo

# Install Bench using `pip`

    sudo pip install -e bench-repo

Once Bench is installed, proceed further to install ERPNext using Bench. Before that, make sure that .config directory is accessible for user bench? If not, do the following command

	sudo chown -R bench:bench .config

# Install ERPNext using Bench

Initialize a bench directory with frappe framework installed. To keep everything tidy, we will work under the /opt/bench directory. Bench will also setup regular backups and auto updates once a day.

    cd /opt/bench
    sudo npm install -g yarn
    bench init erpnext && cd erpnext

# Create a new Frappe site and install ERPNEXT app

    bench new-site --db-name erpnext erp.site.io

The above command will prompt you for the MySQL root password. Provide the password which you have set for the MySQL root user earlier. It will also ask you to set a new password for the administrator account. You will need this password later to log into the administrator dashboard. The option `--db-name` is a database name which MariaDB will create.

Download ERPNext installation files from the remote git repository using Bench.

    bench get-app erpnext https://github.com/frappe/erpnext

Install ERPNext on your newly created site.

    bench --site erp.site.io install-app erpnext

You can start the application immediately to check if the application installed successfully.

    bench start

However, you should stop the execution and proceed further to set up the application for production use.

# Setup Supervisor and Nginx

By default, the ERPNext application listens on port `8000`, not the standard HTTP port `80`. Also, running the built in web server for production use is not recommended as we will be exposing the server to the world. You should use a production web server as a reverse proxy such as Apache or Nginx. We will use Nginx as a reverse proxy as it can be automatically configured using Bench. Bench can automatically generate and install the configuration according to the ERPNext setup.

Although we can start the application using the `'bench start'` command, the execution of ERPNext will stop as soon as you close the terminal. To overcome this issue, you should use Supervisor, which is very helpful in running the application continuously in a production environment. Supervisor is a process control system that enables you to monitor and control a number of processes on Linux operating systems. Once Supervisor is configured, it will automatically start the application at boot time as well as on failures. Bench can automatically configure Supervisor for the ERPNext application.

# Install Supervisor

    sudo apt -y install supervisor

Start Supervisor and enable it to automatically start at boot time.

    sudo systemctl start supervisor
    sudo systemctl enable supervisor

# Setup Bench for production use

    sudo bench setup production bench

The above command may prompt you before replacing the existing Supervisor default configuration file with a new one. Choose `y` to proceed. Bench adds a number of processes to the Supervisor configuration file. The above command will also ask you if you wish to replace the current Nginx configuration with a new one. Enter `y` to proceed. Once Bench has finished installing the configuration, provide other users to execute the files in your home directory of the Bench user.

    chmod o+x /opt/bench/

You can now access the site on http://erp.site.io. And get back to the console to check the status of the processes by running.

    sudo supervisorctl status all

You should see the following output.

    bench@erp-site-io:~/erpnext$ sudo supervisorctl status all
    erpnext-redis:erpnext-redis-cache                 RUNNING   pid 13852, uptime 0:00:54
    erpnext-redis:erpnext-redis-queue                 RUNNING   pid 13851, uptime 0:00:54
    erpnext-redis:erpnext-redis-socketio              RUNNING   pid 13853, uptime 0:00:54
    erpnext-web:erpnext-frappe-web                    RUNNING   pid 13856, uptime 0:00:54
    erpnext-web:erpnext-node-socketio                 RUNNING   pid 13855, uptime 0:00:54
    erpnext-workers:erpnext-frappe-default-worker-0   RUNNING   pid 13862, uptime 0:00:54
    erpnext-workers:erpnext-frappe-long-worker-0      RUNNING   pid 13870, uptime 0:00:54
    erpnext-workers:erpnext-frappe-schedule           RUNNING   pid 13869, uptime 0:00:54
    erpnext-workers:erpnext-frappe-short-worker-0     RUNNING   pid 13875, uptime 0:00:54

To stop all of the ERPNext processes.

    sudo supervisorctl stop all

To start all the ERPNext processes.

    sudo supervisorctl start all

# Setting Up SSL using Let's Encrypt

Let's Encrypt provides free SSL certificates to the users. SSL can be installed manually or automatically through Bench. Bench can automatically install the Let's Encrypt client and obtain the certificates. Additionally, it automatically updates the Nginx configuration to use the certificates.

The domain name which you are using to obtain the certificates from the Let's Encrypt CA must be pointed towards the server. The client verifies the domain authority before issuing the certificates.

Enable DNS multi-tenancy for the ERPNext application.

    bench config dns_multitenant on

Run Bench to set up Let's Encrypt on your site.

    sudo bench setup lets-encrypt erp.site.io
During the execution of the script, the Let's Encrypt client will ask you to temporarily stop the Nginx web server. It will automatically install the required packages and the Let's Encrypt client. The client will prompt you for your email address. You will also need to accept the terms and conditions. Once the certificates have been generated, Bench will also generate the new configuration for Nginx which uses the SSL certificates. You will be asked before replacing the existing configuration. Bench also creates a crontab entry to automatically renew the certificates every month.

Finally, enable scheduler to automatically run the scheduled jobs.

    bench enable-scheduler

You should see this output.

    bench@erp-site-io:~/erpnext$ bench enable-scheduler
    Enabled for erp.site.io

Preview your ERPNext site with SSL at `https://erp.site.io`

# See your database structure and data
For ERPNext site, you can access to database structure and see the data with below command.

    bench@erp-site-io:~/erpnext$ bench mariadb
    Version: 1.17.0
    Chat: https://gitter.im/dbcli/mycli
    Mail: https://groups.google.com/forum/#!forum/mycli-users
    Home: http://mycli.net
    Thanks to the contributor - Darik Gamble
    erp.site.io> show tables;
    +---------------------------------------------------+
    | Tables_in_erpnext                                 |
    +---------------------------------------------------+
    | __Auth                                            |
    | __UserSettings                                    |
    | __global_search                                   |
    | help                                              |
    | tabAbout Us Team Member                           |
    | tabAcademic Term                                  |
    | tabAcademic Year                                  |
    | tabAccount                                        |
    | tabAccounting Period                              |
    | tabActivity Cost                                  |
    | tabActivity Log                                   |
    | tabActivity Type                                  |
    | tabAdditional Salary                              |
    | tabAddress                                        |
    | tabAddress Template                               |
    | tabAgriculture Analysis Criteria                  |
    | tabAgriculture Task                               |
    | tabAllowed To Transact With                       |
    | tabAntibiotic                                     |
    | tabAppointment Type                               |

# Setup help
If you are facing of error on the search box when trying to see help. Use the below command to install the help database.

    bench setup-help

# Remove ERPNext application

ERPNext is the one of application of Frappe framework so you can remove. Using below command, the ERPNext will be removed.

    bench@erp-site-io:~/erpnext$ bench remove-app erpnext

# Dropping your site and remove database
To drop your site, you have to provide the database password which asks from bench. A backup will be created and display location when finished.

    bench@erp-site-io:~/erpnext$ bench drop-site erp.site.io
    Backed up files /opt/bench/erpnext/sites/erp.site.io/private/backups/20180620_115121-erp_site_io-files.tar
    Backed up files /opt/bench/erpnext/sites/erp.site.io/private/backups/20180620_115121-erp_site_io-private-files.tar
    MySQL root password:

Go to list your file system of erpnext directory, you will see a new directory `archived_sites`

    bench@erp-site-io:~/erpnext$ ls
    Procfile  apps  archived_sites  config  env  logs  patches.txt  sites

Drop database `erpnext`

    bench@erp-site-io:~$ mysql -uroot -p -e 'DROP DATABASE erpnext;'

Remove all ERPNext files

    bench@erp-site-io:~$ ls
    bench-repo  erpnext
    bench@erp-site-io:~$ rm -rf erpnext

Reboot the system

    bench@erp-site-io:~$ sudo reboot

# [Note] Stopping Production and starting Development

    cd frappe-bench
    rm config/supervisor.conf
    rm config/nginx.conf
    sudo service nginx stop
    sudo service supervisor stop
    bench setup procfile
    bench start

# [Note] Resolution if has database issue

https://discuss.erpnext.com/t/guide-manual-install-erpnext-on-ubuntu-17-xx-18-xx/36745/3
    
# [Note] Problem Solving: Resolution for error setup

https://discuss.erpnext.com/t/solved-setup-failed-could-not-start-up-error-in-setup/33037/2

# [Note] Conclusion

Once the process has finished, you can access your application at https://erp.site.io. Login with the username Administrator and the password you have set during installation. You will be taken to the desk where you will need to provide information to set ERPNext ERP according to your company. You can now use the application to manage your company.

Congratulations, you have a fully working ERPNext application installed on your Ubuntu 18.04 server.