# Indigo Debian/LXC - Production

How to install [Laws-Africa/Indigo](https://github.com/laws-africa/indigo) in an Debian 11 LXC Container. Note this will install the current stable (17.0.0) version from the repo. Note these steps will also work for a bare-metal Debian 11 installation.

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
 8. [Django](https://www.djangoproject.com) (A rich web platform)
 9. [Indigo](https://github.com/laws-africa/indigo) (A specialized document management system)

## Getting started

We should always start on a clean and updated Debian 11 install, as this includes nano by default, I will be useing nano to edit files, you are free to use your favorite. It is important to know that for some reason doing this via ssh causes issues with executibles so I run this throuh a tty connection in Proxmox. If I figure out the ssh issue I will fix it. Everything here is done as root.

 1. Update Debian:
 ```bash
 apt update && apt upgrade -y
 ```
 
 2. Install the prerequisite packages from apt:
 ```bash
 apt install git build-essential poppler-utils fontconfig xfonts-base xfonts-75dpi postgresql ruby ruby-dev python3-pip python3-dev ghostscript --no-install-recommends -y
 ```
 
 3. Install WKHTMLtoPDF
 ```bash
 wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.bullseye_amd64.deb
 ```
 ```bash
 dpkg -i wkhtmltox_0.12.6.1-2.bullseye_amd64.deb
 ```
 
## Configure Pythons pip and install some requirements

 1. Update pip:
 ```bash
 pip install -U pip
 ```
 
 2. Update setuptools and wheel
 ```bash
 pip install -U setuptools wheel
 ```
 
 3. Install some required PIP Packages:
  ```bash
 pip install gevent==21.8.0 gunicorn==20.1.0 psycopg2-binary==2.9.4
 ```

 ## Create the postgres database:

 For this part we will use the database named indigodb and user named indiguser with password 123Password. Make sure you use a secure username and password, and pass the correct data to the ENV variable below, the correct format for DATABASE_URL is postgres://USER:PASSWORD@HOST:PORT/DBNAME so make sure you update this string accordingly before exporting it to the bashrc file. To create the database and user, as root:
 ```bash
 su - postgres -c 'createdb indigodb'
 ```
 ```bash
 su - postgres -c 'createuser -d -P indigouser'
 ```
 Choose password: 123Password
 
 If your db name, username and password are not the same as the example above, make sure to change the below DATABASE_URL ENV variable accordingly:
  ```bash
 echo 'export DATABASE_URL=postgres://indigouser:123Password@localhost:5432/indigodb' >> ~/.bashrc
 ```
 Enable the changes to the .bashrc file.
 ```bash
 source ~/.bashrc
 ```
 
 ## Now let's install Indigo

 1. Let's clone into the current Indigo Development Branch:
 ```bash
 git clone --branch v17.0.0 --single-branch https://github.com/laws-africa/indigo
 ```
 
 2. Change into the indigo folder (/root/indigo/)
 ```bash
 cd indigo
 ```
 
 3. Make some changes to the settings.py file (/root/indigo/indigo/settings.py):
 First we are going to enable Emails as Background Tasks:
 ```bash
 sed -i "s/'NOTIFICATION_EMAILS_BACKGROUND': False,/'NOTIFICATION_EMAILS_BACKGROUND': True,/" ./indigo/settings.py
 ```
 If relevant to you, enable TLS or SSL by running either of these commands:
 To rely on ENV variable - Testing
 ```bash
 echo "EMAIL_USE_TLS = os.environ.get('DJANGO_EMAIL_USE_TLS', False)" >> ./indigo/settings.py
 ```
 To enable TLS:
 ```bash
 sed -i "/^EMAIL_PORT = int(os.environ.get('DJANGO_EMAIL_PORT', 25))/a EMAIL_USE_TLS = True" ./indigo/settings.py
 ```
 To enable SSL:
 ```bash
 sed -i "/^EMAIL_PORT = int(os.environ.get('DJANGO_EMAIL_PORT', 25))/a EMAIL_USE_SSL = True" ./indigo/settings.py
 ```
 
 4. Install Indigo Python dependencies, including Django:
 ```bash
 pip install -e .
 ```
 
 4, Install Indigo Ruby dependencies:
 First we need to fix the Gemfile and Gemfile.lock due to an issue with incorrect slaw version:
 ```bash
 sed -i "s/slaw (12.0.0)/slaw (13.0.0)/" ./Gemfile.lock
 ```
 ```bash
 sed -i "s/slaw (~> 12.0)/slaw (~> 13.0)/" ./Gemfile.lock
 ```
 ```bash
 sed -i "s/gem 'slaw', '~> 12.0'/gem 'slaw', '~> 13.0'/" ./Gemfile
 ```
 Then install the gems necessary for Indigo:
 ```bash
 gem install bundler
 ```
 ```bash
 bundle install
 ```

## Configure your Indigo Installation:

 1. Create the initial Database schema:
 ```bash
 python3 manage.py migrate
 ```
 
 2. Pass your specific settings to Indigo via ENV Variables:
 Edit tha .bashrc file for your user:
 ```bash
 nano ~/.bashrc
 ```
 Add the following lines, making sure to match your own configuration:
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
 export DJANGO_EMAIL_USE_TLS=true
 ```
 Enable these ENV variables:
 ```bash
 source ~/.bashrc
 ```
 
 3. Generate some SSL Certificates (Gunicorn and Indigo Production require HTTPS):
 ```bash
 openssl req -new -x509 -days 365 -nodes -out /root/server.crt -keyout /root/server.key
 ```
 Enter the appropriate settings as you need them.
 
 2. Import Countries and Languages
 ```bash
 python3 manage.py update_countries_plus
 ```
 ```bash
 python3 manage.py loaddata languages_data.json.gz
 ```
 
 3. Update the Database with any added Migrations (Our changes to settings.py above):
 ```bash
 python3 manage.py makemigrations
 ```
 ```bash
 python3 manage.py migrate
 ```
 ```bash
 
 4. Import available countries from Django-Countries-Plus:
 ```bash
 python3 manage.py update_countries_plus
 ```
 
 5. Import languages maintained by Indigo:
 ```bash
 python3 manage.py loaddata languages_data.json.gz
 ```
 
 6. Compile static files for the webserver:
 ```bash
 python3 manage.py compilescss
 ```
 ```bash
 python3 manage.py collectstatic --noinput -i docs -i \*scss 2>&1
 ```
 
 7. Create a Superuser account
 ```bash
 python3 manage.py createsuperuser
 ```
 Complete with your superuser settings as you need.

## Run the Produciton Server and complete intial configuration

  1. Start the production server with gunicorn:
  ```bash
  gunicorn indigo.wsgi:application -k=gevent -t 600 --certfile=/root/server.crt --keyfile=/root/server.key -b=0.0.0.0:8000 -w=16 --threads 16 --forwarded-allow-ips=* --proxy-allow-from=* --limit-request-line 0
  ```
  To understand each of these arguments:
   --worker-class or -k : What kind of worker (use gevent)
   -t : timeout (use 600ms for now)
   --bind or -b : Bind to address:port (use 0.0.0.0:8000)
   --workers or -w : How many workers (use 8 - 2x cores)
   --threads : How many threads per worker (Use 8 - 2x cores)
   -D : Run as daemon (Background Service)
   --limit-request-line : Length of request line (set to 0)
  
  2. Access your development server via http://your-ip:8000
  
  3. Login with the superuser account created earlier.
  
  4. In the top-right corner click your username and on th edropdown menu select "Site sttings".
  
  5. Under the Indigo API section, add a language.
  
  6. Also under the Indigo API section, add a country, making sure to select a default language.
  
  7. Edit your superuser account and add the country created above as the user's default country.

## Use Supervisor to automate server startup - Testing - Working

Through significant trial and error, some walkthroughs and dumb luck, I managed to get supervisor to autostart gunicorn, so to make this work, do the following:

1. Install Supervisor:
```bash
apt update && apt install supervisor --no-install-receommends -y
```

2. Enter the root folder and create the gunicorn_configuration file that Supervisor will use:
```bash
cd /root
```
```bash
touch gunicorn_configuration
```

3. Edit the newly created gunicorn_configuration file:
```bash
nano gunicorn_configuration
```
And make it look like this (Substiute folder paths, user and group if you are not using root as per this example, make sure your export variables match those used in your ~/.bashrc file, this is necessary for Supervisor which cannot use the ~/.bashrc file):
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
export DJANGO_EMAIL_USE_TLS=true
#Command to run the progam under supeperisor
exec gunicorn --chdir /root/indigo indigo.wsgi:application -k=gevent -t 600 --certfile=/root/server.crt --keyfile=/root/server.key -b=0.0.0.0:8000 -w=16 --threads 16 --forwarded-allow-ips=* --proxy-allow-from=* --limit-request-line 0 --log-level=debug --log-file=-
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
command=/root/gunicorn_configuration
user=root
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/root/gunicorn-error.log
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

## Use Crontab to automate background tasks to run - Testing - Not Working

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
/usr/bin/python3
```
3. In our example, Indigo is installed in /root/indigo, so to determine our cron script, it will look like this:
{Cron Timing} {Python 3 Path} {Indigo Path} {manage.py switches}:
```bash
*/30 * * * * /usr/bin/python3 /root/indigo/manage.py process_tasks --duration 60
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

2. Now, since the only "custom" data is stored in AWS, your Postgres DB and your settings.py file, we delete the indigo folder:
```bash
rm -rf indigo/
```

3. And now we clone the latest version of indigo from GitHub (Substitute v17.0.0 for the newer one):
```bash
git clone --branch v17.0.0 --single-branch https://github.com/laws-africa/indigo
```

4. We need to now install any updated dependencies which is similar to how we originally installed Indigo:
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

5. Now we need to reconfigure our settings.py file (With the same settings we used in our initial installation, good thing we have easy commands to do that:
 ```bash
 sed -i "s/'NOTIFICATION_EMAILS_BACKGROUND': False,/'NOTIFICATION_EMAILS_BACKGROUND': True,/" ./indigo/settings.py
 ```
 ```bash
 echo "EMAIL_USE_TLS = os.environ.get('DJANGO_EMAIL_USE_TLS', False)" >> ./indigo/settings.py
 ```

6. Update the relevant database fields and rebuild static files:
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

7. Re-enable the supervisor app:
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

1. Automate Gunicorn to start at system boot, no clue how to get this to properly work, if someone could assist here it would be great (Currently testing with Supervisor).

2. Updating, while this could work with a simple GIT Fetch, the fact that you NEED to change settings.py makes this more difficult. Will optimize this once I get time. For now I have added manual steps which essentially require reinstalling Indigo from the latest git.

3. Get Backtround Tasks to play along with Cron, not really working as expected.
