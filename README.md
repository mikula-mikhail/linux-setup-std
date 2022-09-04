# linux-setup-std
Ubuntu/Debian server setup

## Setup SSH

```
sudo apt-get update ; \
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make
```

Configure SSH:

```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    PermitRootLogin no
    PasswordAuthentication no
```

Restart SSH server, change `www` user password:

```
sudo service ssh restart
sudo passwd www
```

## Init — must-have packages

```
sudo apt-get install -y tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python-imaging python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```

## ZSH

```
sudo apt-get install -y zsh

sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

Configure some needed aliases:

```
vim ~/.zshrc
    alias cls="clear"
```

## Install python 3.7

mkdir ~/code

Build from source python 3.7, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz ; \
tar xvf Python-3.7.* ; \
cd Python-3.7.3 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall
```

Now python3.7 in `/home/www/.python/bin/python3.7`. Update pip:

```
sudo /home/www/.python/bin/python3.7 -m pip install -U pip
```

create and activate Python virtual environment:

```
cd code
git pull project_git
cd project_dir
python3.7 -m venv env
. ./env/bin/activate
```

## Install and configure PostgreSQL

Install PostgreSQL 11 and configure locales.

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
RELEASE=$(lsb_release -cs) ; \
echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
sudo apt update ; \
sudo apt -y install postgresql-11 ; \
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locales
```

Add locales to `/etc/profile`:

```
sudo vim /etc/profile
    export LANGUAGE=ru_RU.UTF-8
    export LANG=ru_RU.UTF-8
    export LC_ALL=ru_RU.UTF-8
```

Change `postges` password, create clear database named `dbms_db`:

```
sudo passwd postgres
su - postgres
export PATH=$PATH:/usr/lib/postgresql/11/bin
createdb --encoding UNICODE dbms_db --username postgres
exit
```

Create `dbms` db user and grand privileges to him:

```
sudo -u postgres psql
postgres=# ...
create user dbms with password 'some_password';
ALTER USER dbms CREATEDB;
grant all privileges on database dbms_db to dbms;
\c dbms_db
GRANT ALL ON ALL TABLES IN SCHEMA public to dbms;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public to dbms;
GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to dbms;
CREATE EXTENSION pg_trgm;
ALTER EXTENSION pg_trgm SET SCHEMA public;
UPDATE pg_opclass SET opcdefault = true WHERE opcname='gin_trgm_ops';
\q
exit
```

test connection. Create `~/.pgpass` with login and password to db for fast connect:

```
vim ~/.pgpass
	localhost:5432:dbms_db:dbms:some_password
chmod 600 ~/.pgpass
psql -h localhost -U dbms dbms_db
```

Run SQL dump, if you have:

```
psql -h localhost dbms_db dbms  < dump.sql
```

## Install and configure `supervisor`

recommended way is using `Systemd` instead of `supervisor`

```
sudo apt install supervisor

vim /home/www/code/project/bin/start_gunicorn.sh
	#!/bin/bash
	source /home/www/code/project/env/bin/activate
	source /home/www/code/project/env/bin/postactivate
	exec gunicorn  -c "/home/www/code/project/gunicorn_config.py" project.wsgi

chmod +x /home/www/code/project/bin/start_gunicorn.sh

vim project/supervisor.salesbeat.conf
	[program:www_gunicorn]
	command=/home/www/code/project/bin/start_gunicorn.sh
	user=www
	process_name=%(program_name)s
	numprocs=1
	autostart=true
	autorestart=true
	redirect_stderr=true
```

`Gunicorn` config:
`pip install gunicorn`
gunicorn_config.py
```
command = '/home/www/code/project/env/bin/gunicorn'
pythonpath = '/home/www/code/project/project'
bind = '127.0.0.1:8001'
workers = 3
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=project.settings'
```
go to project folder
`mkdir bin`
edit bin/start_gunicorn.sh
```
#!/bin/bash
source /home/<user>/<project_dir>/env/bin/activate
exec gunicorn -c "/home/<user>/<project_dir>/gunicorn_config.py" <project_config>.wsgi
```
make file executable & run:
```
chmod +x bin/start_gunicorn.sh
. ./bin/start_gunicorn.sh
```

## MariaDB

```
sudo apt install -y mariadb-server mariadb-client
sudo mysql_secure_installation
```

Login with `root` user is available just with `sudo`:

```
sudo mariadb -u root
```

So, create database and user and give him privileges to new database:

```
sudo mariadb -u root
CREATE DATABASE new_db_name COLLATE 'utf8_general_ci';
CREATE USER new_db_user IDENTIFIED BY 'some_password';
GRANT ALL privileges ON new_db_name .* TO new_db_user;
```

## Apache, PHP

Install Apache, create `code` directory in user home directory:

```
sudo apt install -y apache2 apache2-utils
cd
mkdir code
cd code
mkdir newproject
sudo chown www-data:www-data /home/www/code/ -R
```

Now install PHP:

```
sudo apt install -y php7.0 libapache2-mod-php7.0 php7.0-mysql php-common php7.0-cli php7.0-common php7.0-json php7.0-opcache php7.0-readline php7.0-mbstring php7.0-xml php7.0-gd
```

Enable `mod_php` in Apache:

```
sudo a2enmod php7.0
sudo systemctl restart apache2
```

Configure Apache:

```
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/newproject.conf

sudo vim /etc/apache2/sites-available/newproject.conf
```

Change or append lines:

```
        DocumentRoot /home/www/code/newproject
        ServerName newproject.your-site.ru
```

Enable site config in Apache, enable `mod_rewrite`:

```
sudo a2ensite newproject.conf

sudo a2enmod rewrite
```

Configure Apache for `.htaccess` files:

```
sudo vim /etc/apache2/apache2.conf
```

Change lines:

```
<Directory />
    Options FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>
```

Restart Apache:

```
sudo service apache2 restart
```
