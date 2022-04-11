# Indigo-LXC

How to install [Laws-Africa/Indigo](https://gitbub.com/laws-africa/indigo) in an LXC Container for production use.

All credit and atribution to [Laws-Africa](https://github.com/laws-africa/) for their excellent work on the Indigo Platform.

## What is Indigo

If you are asking that here, I think you are starting at the wrong place and I suggest you start at [their official Github page](https://github.com/laws-africa/indigo) to gain a proper understanding of how it works. If you are just looking to deploy it, follow their official instructions. This How-to is written specifically to have a production setup running in a Debian 11 LXC Container (However other distros may work too) running on Proxmox.

## What are we installing

 1. [Debian](https://www.debian.org/) 11 (GNU/Linux OS)
 2. [Postgres](https://www.postgresql.org/) (Database Server)
 3. [Python3](https://www.python.org/) (Scripting Language)
 4. [WKHMTLtoPDF](https://wkhtmltopdf.org/) (PDF Creation - Not currently working)
 5. [Poppler](https://poppler.freedesktop.org/) (PDF Rendering Library)
 7. [Ruby](https://www.ruby.org/) (Another popular Scripting Langauge)
 8. [RBENV](https://github.com/rbenv) (Simplified Ruby Management for Linux)
 9. [Django](https://www.djangoproject.com) (A rich web platform)
 10. [Indigo](https://github.com/laws-africa/indigo) (A specialized document management system)

## Getting started

We should always start on a clean and updated Debian 11 install, as this includes nano by default, I will be useing nano to edit files, you are free to use your favorite. It is important to know that for some reason doing this via ssh causes issues with executibles so I run this throuh a tty connection in Proxmox. If I figure out the ssh issue I will fix it. Everything here is done as root.

 1. Update Debian:
 ```bash
 apt update && apt upgrade -y
 ```
 
 2. Install the prerequisite packages:
 ```bash
 apt install git curl libssl-dev libreadline-dev zlib1g-dev autoconf bison build-essential libyaml-dev libreadline-dev libncurses5-dev libffi-dev libgdbm-dev xfonts-base xfonts-75dpi fontconfig xfonts-encodings xfonts-utils poppler-utils postgresql python3-pip libpq-dev libpoppler-dev sqlite3 libsqlite3-dev wkhtmltopdf libbz2-dev  --no-install-recommends -y
 ```
 
 3. Create the postgres database:

 For this part we will use the database named indigo and user named indigo with password indigo. If you want to use anything else, you will need to also change the mapped fields in root/indigo/indigo/setings.py related to:
 ```bash
 nano /root/indigo/indigo/settings.py
 ```
 Edit the following line with your configuration (This could be better done through using the correct ENV variables and Django dj_database_url, will look into this):
 ```bash
 db_config = dj_database_url.config(default='postgres://indigo:indigo@localhost:5432/indigo')
 ```
 To create the database, as root:
 ```bash
 su - postgres -c 'createuser -d -P indigo'
 ```
 Either make the password indigo or see the notes above regarding settings.py
 
 4. Set your ENV Variables that Indigo requires:
 
 Edit the bashrc file:
 ```bash
 nano ~/.basrch
 ```
 And add the following fields (The parts in {Curly Brackets} need to match your system, note that AWS S3 is required for storing files:
 ```bash
 export DJANGO_DEBUG=false
 export DJANGO_SECRET_KEY={Some Random Characters}
 export AWS_ACCESS_KEY_ID={Your AWS Key}
 export AWS_SECRET_ACCESS_{Your AWS Access ID}
 export AWS_S3_BUCKET={The name of your AWS Bucket}
 export SUPPORT_EMAIL={Your admin email address}
 export DJANGO_DEFAULT_FROM_EMAIL={The email address Indigo will send mail from}
 export DJANGO_EMAIL_HOST={Your SMTP Configaration}
 export DJANGO_EMAIL_HOST_USER={Your SMTP Configaration}
 export DJANGO_EMAIL_HOST_PASSWORD={Your SMTP Configaration}
 export DJANGO_EMAIL_PORT={Your SMTP Configaration}
 export INDIGO_ORGANISATION='{Your Organization Name}'
 export RECAPTCHA_PUBLIC_KEY={Your Google Recaptcha Key}
 export RECAPTCHA_PRIVATE_KEY={Your Google Recaptcha Key}
 export GOOGLE_ANALYTICS_ID={Your Google Analytics ID}
 ```
 Enable this
 ```bash
 source ~/.bashrc
 ```
 
 
## Install Rbenv and Ruby version 2.7.2 (Officially current Ruby version of the Indigo Project):

 1. Install Rbenv
 
 The nice perople at rbenv have created a curl script we can use:
 ```bash
 curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
 ```

 2. After the installation, rbenv needs some configurations to system ENV, so:
 ```bash
 echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
 ```
 ```bash
 echo 'eval "$(rbenv init -)"' >> ~/.bashrc
 ```
 
 3. Now enable the local ENV Vars:
 ```bash
 source ~/.bashrc
 ```

 4. Test rbenv:
 ```bash
 rbenv -v
 ```
 Which should return your rbenv version
 
 5. Install Ruby 2.7.2
 ```bash
 rbenv install 2.7.2
 ```
 
 5. Set Ruby 2.7.2 as your global Ruby version:
 ```bash
 rbenv global 2.7.2
 ```
 
 6. Test your Ruby installation:
 ```bash
 ruby -v
 ```
 Which should show Ruby version 2.7.2
 
### Configure Pythons PIP and install some requirements

 1. Update PIP:
 ```bash
 pip install --upgrade pip
 ```
 
 2. Install some required PIP Packages:
 
 Wheel
 ```bash
 pip install wheel
 ```
 Setuptools (Required by PsycoPG2)
 ```bash
 pip install -U pip setuptools
 ```
 Gevent (Required for Indigo to work with Gunicorn)
 ```bash
 pip install gevent==21.8.0
 ```
 Gunicorn (A Lightweight webserver)
 ```bash
 pip install gunicorn==20.1.0
 ```
 ```bash
 pip install psycopg2==2.8.6
 ```

## Now let's install Indigo

 1. Let's clone into the current Indigo Master Branch:
 ```bash
 git clone https://github.com/laws-africa/indigo
 ```
 
 2. Change into the indigo folder (/root/indigo/)
 ```bash
 cd indigo
 ```
 
 3. Install Indigo Python dependencies, including Django 2.2:
 ```bash
 pip install -e .
 ```
 
 4, Install Indigo Ruby dependencies:
 ```bash
 gem install bundler
 ```
 ```bash
 bundle install
 ```

## Configure your Indigo Installation:

 1. Indigo Settings
 
 Indigo stores it's settings in a Django settings file located at /root/indigo/indigo/settings.py, however since we are already in the /root/indigo folder:
 ```bash
 nano ./indigo/settings.py
 ```
 This file is reasonably well documented, however I suggest at least enabling Background Emails:
 ```bash
 'NOTIFICATION_EMAIL_BACKGROUND': True,
 ```
 
 2. Create / Update Database Tables
 ```bash
 python3 manage.py migrate
 ```
 
 3. Import Countries and Languages
 ```bash
 python3 manage.py update_countries_plus
 ```
 ```bash
 python3 manage.py loaddata languages_data.json.gz
 ```
 
 4. Create a Superuser account
 ```bash
 python3 manage.py createsuperuser
 ```
 
 5. Compile static files (Otherwise Gunicorn won't work):
 ```bash
 python3 manage.py compilescss
 ```
 ```bash
 python3 manage.py collectstatic --noinput -i docs -i \*scss 2>&1
 ```
 
 6. Create SSL Certificates (Could be done with certbot, but would also require changing the Gunicorn String and then requires public facing server):
 ```bash
 openssl req -new -x509 -days 365 -nodes -out server.crt -keyout server.key
 ```
 
## Start your Indigo Server using Gunicorn

I use Gunicorn on my LXC deployment which hosts the page internally, publicly it sits behind an NginX reverse proxy so this works for me. Feel free to use NginX along with Gunicorn.

The following script will execute Gunicorn and start listening for https traffic on port 8000 (Note, it must be run from the /root/indigo folder):
```bash
gunicorn indigo.wsgi:application -k=gevent -t 600 --certfile=/root/indigo/server.crt --keyfile=/root/indigo/server.key -b=0.0.0.0:8000 -w=16 --threads 16 --forwarded-allow-ips=* --proxy-allow-from=* --limit-request-line 0
```
To understand each of these arguments:
--worker-class or -k : What kind of worker (use gevent)
-t : timeout (use 600ms for now)
--bind or -b : Bind to address:port (use 0.0.0.0:8000)
--workers or -w : How many workers (use 8 - 2x cores)
--threads : How many threads per worker (Use 8 - 2x cores)
-D : Run as daemon (Background Service)
--limit-request-line : Length of request line (set to 0)

You can now connect to your Indigo Server on https://your_ip:8000

## Use Supervisor to automate server startup (Testing)

Through significant trial and error, some walkthroughs and dumb luck, I managed to get supervisor to autostart gunicorn, so to make this work, do the following:

1. Install Supervisor
```bash
apt update && apt install supervisor -y
```
2. Enter the indigo installation folder to create a gunicorn_config file
```bash
cd /root/indigo
```
```bash
touch gunicorn_configuration
```
```bash
nano gunicorn_configuration
```
And make it look like this (Substiute folder paths if you are not using root):
```bash
#!/bin/bash
NAME="Indigo"  #Django application name
DIR=/root/indigo   #Directory where project is located
USER=root   #User to run this script as
GROUP=root  #Group to run this script as
DJANGO_SETTINGS_MODULE=indigo.settings   #Which Django setting file should use
DJANGO_WSGI_MODULE=indigo.wsgi           #Which WSGI file should use
LOG_LEVEL=debug
cd $DIR
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DIR:$PYTHONPATH
#Command to run the progam under supeperisor
exec gunicorn --chdir /root/indigo indigo.wsgi:application -k=gevent -t 600 --certfile=/root/indigo/server.crt --keyfile=/root/indigo/server.key -b=0.0.0.0:8000 -w=16 --threads 16 --forwarded-allow-ips=* --proxy-allow-from=* --limit-request-line 0 --log-level=debug --log-file=-
```
3. Make this file executable:
```bash
chmod u+x gunicorn_configuration
```
4. Configure Supervisor to start with systemd
```bash
systemctl enable supervisor
```
```bash
systemctl start supervisor
```
5. Create a config file for Supervisor to use:
```bash
touch /etc/supervisor/conf.d/indigo.conf
```
```bash
nano /etc/supervisor/conf.d/indigo.conf
```
And make it look like this:
```bash
[program:indigo]
command=/root/indigo/gunicorn_configuration
user=root
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/root/indigo/gunicorn-error.log
```
6. Enable and start your new Supervisor app:
```bash
supervisorctl reread
```
```bash
supervisorctl update
```
```bash
supervisorctl restart indigo
```
And voila, Supervisor will now start your Indigo Gunicorn server at bootup

## To Do!

1. Automate Gunicorn to start at system boot, no clue how to get this to properly work, if someone could assist here it would be great (Currently testing)

2. Updating, while this could work with a simple GIT Fetch, the fact that you NEED to change settings.py makes this more difficult. Will optimize this once I get time.

 
