# Indigo-LXC-Testing Branch

How to install [Laws-Africa/Indigo](https://github.com/laws-africa/indigo) in an LXC Container. Note this will install latest development from the repo. For production use, refer to the current stable release (17.0.0 at the time of publishing).

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
 9. [PyEnv](https://github.com/pyenv/pyenv) (Simplified Python Management for Linux)
 10. [Django](https://www.djangoproject.com) (A rich web platform)
 11. [Indigo](https://github.com/laws-africa/indigo) (A specialized document management system)
 12. [NodJS](https://nodejs.org) (A JavaScript Runtime)
 13. [NPM](https://www.npmjs.com) (Java Pacakge Repo)
 14. [SASS](https://sass-lang.com) (CSS Extension Language)

## Getting started

We should always start on a clean and updated Debian 11 install, as this includes nano by default, I will be useing nano to edit files, you are free to use your favorite. It is important to know that for some reason doing this via ssh causes issues with executibles so I run this throuh a tty connection in Proxmox. If I figure out the ssh issue I will fix it. Everything here is done as root.

 1. Update Debian:
 ```bash
 apt update && apt upgrade -y
 ```
 
 2. Install the prerequisite packages:
 ```bash
 apt install curl libcurl4 git build-esential zlib1g-dev nodejs npm poppler-utils fontconfig xfonts-75dpi xfonts-base postgresql --no-install-recommends -y
 ```
 
## Install WKHTMLtoPDF

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.bullseye_amd64.deb
```
```bash
dpkg -i wkhtmltox_0.12.6.1-2.bullseye_amd64.deb
```
 
## Install Rbenv and Ruby version 2.7:

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
 
 5. Install Ruby 2.7.6
 ```bash
 rbenv install 2.7.6
 ```
 
 5. Set Ruby 2.7.6 as your global Ruby version:
 ```bash
 rbenv global 2.7.6
 ```
 
 6. Test your Ruby installation:
 ```bash
 ruby -v
 ```
 Which should show Ruby version 2.7.6

### Install PyEnv and Python 3.8

Due to the requirements of django-background-task, we are limited to Python 3.7.

 1. Install PyEnv
 
 Use the Curl script:
 ```bash
 curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
 ```

 2. After the installation, rbenv needs some configurations to system ENV, so:
 ```bash
 nano ~/.bashrc
 ```
 and add the following at the end of the file:
 ```bash
 export PYENV_ROOT="$HOME/.pyenv"
 command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
 eval "$(pyenv init -)"
 ```
 
 3. Now enable the local ENV Vars:
 ```bash
 source ~/.bashrc
 ```

 4. Test pyenv:
 ```bash
 pyenv -v
 ```
 Which should return your pyenv version
 
 5. Install Python 3.8.15
 ```bash
 pyenv install 3.8.15
 ```
 
 5. Set Python 3.8.14 as your global Python version:
 ```bash
 pyenv local 3.8.15
 ```
 
 6. Test your Python installation:
 ```bash
 python --version
 ```
 Which should show Python version 3.8.15
 
### Install SASS

 Install SASS
 ```bash
 npm install -g sass
 ```

### Configure Pythons PIP and install some requirements

 1. Update PIP, setuptools and wheel:
 ```bash
 pip install --upgrade pip setuptools wheel
 ```
 
 2. Install some required PIP Packages:
 
 Gevent (Required for Indigo to work with Gunicorn)
 ```bash
 pip install gevent==21.8.0
 ```
 Gunicorn (A Lightweight webserver)
 ```bash
 pip install gunicorn==20.1.0
 ```
 PsycoPG2 (A database library)
 ```bash
 pip install psycopg2-binary==2.9.4
 ```
 ### Create the postgres database:

 For this part we will use the database named indigo and user named indigo with password indigo. If you want to use anything else, you will need to also change the DATABASE_URL variable below:
 The correct format for DATABASE_URL as postgres://USER:PASSWORD@HOST:PORT/DBNAME so if you change this, note that you should also substite the relevant commands below. To create the database and user, as root:
 ```bash
 su - postgres -c 'createdb indigo'
 ```
```bash
su - postgres -c 'createuser -d -P indigo'
```
 If your db name, username and password are not indigo, make sure to configure the DATABASE_URL ENV variable  as follows:
  ```bash
 echo 'export DATABASE_URL={postgres://indigo:indigo@localhost:5432/indigo}' >> ~/.bashrc
 ```
 Enable this
 ```bash
 source ~/.bashrc
 ```
 
 ## Now let's install Indigo

 1. Let's clone into the current Indigo Master Branch:
 ```bash
 git clone --branch v17.2.0 --single-branch https://github.com/laws-africa/indigo
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
 See later in this tutorial to configure crontab to automate background tasks.
 
 2. Create / Update Database Tables
 ```bash
 python manage.py migrate
 ```
 
 3. Import Countries and Languages
 ```bash
 python manage.py update_countries_plus
 ```
 ```bash
 python manage.py loaddata languages_data.json.gz
 ```
 
 4. Create a Superuser account
 ```bash
 python manage.py createsuperuser
 ```
 
 5. Compile static files (Otherwise Gunicorn won't work):
 ```bash
 python manage.py compilescss
 ```
 ```bash
 python manage.py collectstatic --noinput -i docs -i \*scss 2>&1
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

### Set your ENV Variables that Indigo requires for development:
 
 Edit the bashrc file:
 ```bash
 nano ~/.bashrc
 ```
 And add the following fields (The parts in {Curly Brackets} need to match your system, note that AWS S3 is required for storing files
 ```bash
 export DJANGO_DEBUG=false
 export DJANGO_SECRET_KEY={Some Random Characters}
 export AWS_ACCESS_KEY_ID={Your AWS Key}
 export AWS_SECRET_ACCESS_KEY={Your AWS Access ID}
 export AWS_S3_BUCKET={The name of your AWS Bucket}
 export SUPPORT_EMAIL={Your admin email address}
 export DJANGO_DEFAULT_FROM_EMAIL={The email address Indigo will send mail from}
 export DJANGO_EMAIL_HOST={Your SMTP Configaration}
 export DJANGO_EMAIL_HOST_USER={Your SMTP Configaration}
 export DJANGO_EMAIL_HOST_PASSWORD={Your SMTP Configaration}
 export DJANGO_EMAIL_PORT={Your SMTP Configaration}
 export INDIGO_ORGANISATION='{Your Organization Name}'
 export INDIGO_URL={Indogo Official URL}
 export RECAPTCHA_PUBLIC_KEY={Your Google Recaptcha Key}
 export RECAPTCHA_PRIVATE_KEY={Your Google Recaptcha Key}
 export GOOGLE_ANALYTICS_ID={Your Google Analytics ID}
 
 ```
 Enable this
 ```bash
 source ~/.bashrc
 ```

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
And make it look like this (Substiute folder paths, user and group if you are not using root as per this example):
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

## Use Crontab to automate background tasks to run (Testing)

Cron is used as an easy solution to run the manage.py process_tasks instruction. If you want to change how often it is done, you can use [crontab guru](https://crontab.guru/) to calculate the timings. To change the duration, the --duration switch is used to stop the task after x seconds. For this example we will run it every 30 minutes with a 60 second duration.

1. Determine the format for the Cron timings with [crontab-guru](https://crontab.guru/) to run every 30 minutes:
```bash
*/30 * * * *
```
2. Determine the folder where your Python 3 executible is stored:
```bash
which python
```
Which should return something like:
```bash
/root/.pyenv/shims/python
```
3. In our example, Indigo is installed in /root/indigo, so to determine our cron script, it will look like this:
{Cron Timing} {Python 3 Path} {Indigo Path} {manage.py switches}:
```bash
*/30 * * * * /root/.pyenv/shims/python /root/indigo/manage.py process_tasks --duration 60
```
4. Now we simply edit the crontab file with this instruction:
```bash
crontab -e
```
And select nano as editor.
At the bottom of the file we add our expression, from the start of this step:
```bash
*/30 * * * * /root/.pyenv/shims/python /root/indigo/manage.py process_tasks --duration 60
```
Save the file with ctrl+x and enter.

Now, Cron should run the process_tasks instruction every 30 minutes for a duration of 60 seconds.

## Manual Update of Indigo (Very long-winded, essentially reinstalling, anyone want to add an easier way, please do):

I am assuming that you are operting from your root folder and not the indigo installation folder (/root/). This might break your indigo installation, do not proceed unless you have a current backup as these steps are destructive.

1. Stop the indigo app in supervisor (If you are using the automated method for booting indigo in this tutorial):
```bash
supervisorctl stop indigo
```
2. Let's backup our configurations, this will make life easier later:
```bash
cp ./indigo/indigo/settings.py settings.py.bak
```
```bash
cp ./indigo/server.crt server.crt.bak
```
```bash
cp ./indigo/server.key server.key.bak
```
```bash
cp ./indigo/gunicorn_configuraton gunicorn_configuration.bak
```
3. Now, since the only "custom" data is stored in AWS, your Postgres DB and your settings.py file, we delete the indigo folder:
```bash
rm -rf indigo/
```
4. And now we clone the latest version of indigo from GitHub:
```bash
git clone https://github.com/laws-africa/indigo
```
5. We need to now install any updated dependencies which is similar to how we originally installed Indigo:
```bash
cd indigo
```
```bash
pip install -e .
```
```bash
gem install bundler
```
```bash
bundle install
```
6. Now we need to reconfigure our settings.py file (With the same settings we used in our initial installation, good thing we made a backup of this file to reference. Don't replace the new settings.py file with the old one, this might break some changes that the developers introduced.
```bash
nano ./indigo/settings.py
```
And edit the file noting to change the specific settings you did in your initial configuration above. This is important.
7. Return to the root folder:
```bash
cd
```
8. Copy back all your backed up files, except settings.py:
```bash
cp server.crt.bak ./indigo/server.crt
```
```bash
cp server.key.bak ./indigo/server.key
```
```bash
cp gunicorn_configuration.bak ./indigo/gunicorn_configuration
```
9. Go back to the indigo folder:
```bash
cd indigo
```
10. Make the gunicorn_configuration file executable again:
```bash
chmod u+x gunicorn_configuration
```
11. Update the relevant database fields and rebuild static files:
```bash
python manage.py makemigrations
```
```bash
python manage.py migrate
```
```bash
python manage.py update_countries_plus
```
```bash
python manage.py loaddata languages_data.json.gz
```
```bash
python manage.py compilescss
```
```bash
python manage.py collectstatic --noinput -i docs -i \*scss 2>&1
```
12. Re-enable the supervisor app:
```bash
supervisorctl reread
```
```bash
supervisorctl update
```
```bash
supervisorctl restart indigo
```
And if all worked well, you now have an up-to-date installation of Indigo.

## To Do!

1. Automate Gunicorn to start at system boot, no clue how to get this to properly work, if someone could assist here it would be great (Currently testing)

2. Updating, while this could work with a simple GIT Fetch, the fact that you NEED to change settings.py makes this more difficult. Will optimize this once I get time. For now I have added manual steps which essentially require reinstalling Indigo from the latest git.

3. Investigate using Python dotenv to parse ENV variables for Indigo from a .env file, this could open the door to a solution that doesn't require modifying settings.py if Laws-Africa would be willing migrate more settings to os.environ.get.

 
