# How to install taiga on uberspace 7


## Steps:
### 1. Config and parameters

Make sure to have a good configuration machine and prepare a list of parameters that will be frequently used during installation.
As suggested by taiga these are :

```
    IP: 80.88.23.45

    Hostname: example.com (which points to 80.88.23.45)

    Username: isabell

    System memory: >=1GB (needed for compilation of lxml)

    Working directory: /home/taiga/ (default for user taiga)
```

### 2. Install postgres and create database

2.1 Download and extract the source code
```shell
[isabell@stardust ~]$ mkdir ~/postgres/
[isabell@stardust ~]$ cd ~/postgres/
[isabell@stardust ~]$ curl -O https://download.postgresql.org/pub/source/v9.6.10/postgresql-9.6.10.tar.gz
[isabell@stardust ~]$ tar -xvzf ~/postgres/postgresql-9.6.10.tar.gz
```

2.2 Source Code Configuration, Compiling and Installation

```shell
[isabell@stardust ~]$ cd ~/postgres/postgresql-9.6.10
[isabell@stardust ~]$ ./configure --prefix=$HOME/opt/postgresql/ --with-python PYTHON=/usr/bin/python3.6 --without-readline
[isabell@stardust ~]$ make world
[isabell@stardust ~]$ make install-world
```
2.3 Update `~/.bashrc` so that postgres tools are in the path. 

```shell
[isabell@stardust ~]$ vi ~/.bashrc
```

Append the following code to `~/.bashrc`

**Warning:** Replace ``isabell`` with your username! 

```
export HOME=/home/isabell
export PATH=$HOME/opt/postgresql/bin/:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/opt/postgresql/lib
export PGPASSFILE=$HOME/.pgpass
```

Reload the .bash_profile with:

```shell
[isabell@stardust ~]$ source ~/.bashrc
```

Run `psql --version` to verify the installation so far:

```shell
[isabell@stardust ~]$ psql --version
psql (PostgreSQL) 9.6.10
```

2.4 Initialize database

2.4.1 Create the file `~/.pgpass`

```shell
[isabell@stardust ~] $ vi ~/.pgpass
```

 with the following content. Change `isabell` with your user name and password field as per your liking

```
#hostname:port:database:username:password (min 64 characters)
*:*:*:isabell:1234567890123456789012345678901234567890123456789012345678901234
```

2.4.2

```shell
 [isabell@stardust ~]$ chmod 0600 ~/.pgpass
 [isabell@stardust ~]$ cp ~/.pgpass ~/pgpass.temp
```
2.4.3 Edit the pgpass.temp file 

```shell
[isabell@stardust ~] $ vi ~/pgpass.temp
```

to contain only the password.

Check with:

```shell
[isabell@stardust ~]$ cat ~/pgpass.temp
1234567890123456789012345678901234567890123456789012345678901234

```
2.4.4 Initialize DB

Change `isabell` with your user name 

```sh
[isabell@stardust ~]$ initdb --pwfile="/home/isabell/pgpass.temp" --auth=md5 -E UTF8 -D ~/opt/postgresql/data/


The files belonging to this database system will be owned by user "".
This user must also own the server process.
The database cluster will be initialized with locale "de_DE.UTF-8".
The default text search configuration will be set to "german".
Data page checksums are disabled.
creating directory /home/isabell/opt/postgresql/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Success. You can now start the database server using:
pg_ctl -D /home/isabell/opt/postgresql/data/ -l logfile start
```

2.4.5 Remove unused `~/pgpass.temp`

```shell
 [isabell@stardust ~]$ rm ~/pgpass.temp
```

2.4.6 Obtain a free port to be used for DB port number

```shell
[isabell@stardust ~]$ FREEPORT=$(( $RANDOM % 4535 + 61000 )); ss -ln src :$FREEPORT | grep $FREEPORT && echo "try again" || echo $FREEPORT
63743
```
63743 => this is the DB port you need for the database

2.4.7 Update  ~/.bashrc

```shell
[isabell@stardust ~]$ vi ~/.bashrc
```

Add the following to ~/.bashrc and replace the port number with the one you wrote down earlier

```
 export PGHOST=localhost
 export PGPORT=63743
```

and then load the updated `~/.bashrc` by

```shell
[isabell@stardust ~]$ source ~/.bashrc
```



2.4.8  Edit ``~/opt/postgresql/data/postgresql.conf`` and set the key values `listen_adresses` ,
`port` and `unix_socket_directories`:

```shell
[isabell@stardust ~]$ vi ~/opt/postgresql/data/postgresql.conf 
```

**warning:** Replace the port number with the one you wrote down earlier and replace ``isabell`` with your username!

```
#------------------------------------------------------------------------------

# CONNECTIONS AND AUTHENTICATION

#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = 'loclahost'         # what IP address(es) to listen on;
                                       # comma-separated list of addresses;
                                       # defaults to 'localhost'; use '*' for all
                                       # (change requires restart)
port = 63743                           # (change requires restart)
max_connections = 100                  # (change requires restart)
#superuser_reserved_connections = 3    # (change requires restart)
unix_socket_directories = '/home/isabell/tmp'      # comma-separated list of directories

                                       # (change requires restart)
#unix_socket_group = ''                # (change requires restart)
#unix_socket_permissions = 0777        # begin with 0 to use octal notation
                                       # (change requires restart)
#bonjour = off                         # advertise server via Bonjour
                                       # (change requires restart)
#bonjour_name = ''                     # defaults to the computer name
                                       # (change requires restart)

```


2.4.9 Setup service

Create ``~/etc/services.d/postgresql.ini`` 

```shell
[isabell@stardust ~]$ vi ~/etc/services.d/postgresql.ini
```

with the following content:

**warning:** Replace ``isabell`` with your username! (Two times)

```
[program:postgresql]
command=/home/isabell/opt/postgresql/bin/postgres -D /home/isabell/opt/postgresql/data/
autostart=yes
autorestart=yes

```

2.4.10 Update supervisor

```shell
[isabell@stardust ~]$ supervisorctl reread
postgresql: available
[isabell@stardust ~]$  supervisorctl update
postgresql: added process group
[isabell@stardust ~]$  supervisorctl status
postgresql                       RUNNING   pid 15477, uptime 0:00:07
[taiga@dysnomia ~]$
```



2.4.11 Create user and database

Replace the port number with the one you wrote down earlier

```shell
[isabell@stardust ~]$ createuser -h localhost -p 63743 taiga
[isabell@stardust ~]$ createdb -h localhost -p 63743  taiga -O taiga --encoding='utf-8' --locale=en_US.utf8 --template=template0
```



### 3. Setup python and Django

why pip3.6 ?? Because we've use /usr/bin/python3 during installation of postgres and it refers to python3.6

3.1 Create virtualenv and install requirements

```shell
[isabell@stardust ~]$ pip3.6 install virtualenvwrapper --user 
[isabell@stardust ~]$ source ~/.bash_profile
[isabell@stardust ~]$ VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3.6
[isabell@stardust ~]$ source ~/.local/bin/virtualenvwrapper_lazy.sh 
[isabell@stardust ~]$ mkvirtualenv -p /usr/bin/python3.6 taiga
```

3.1.1 Load the virtualenv

```shell
[isabell@stardust ~]$ source ~/.virtualenvs/taiga/bin/activate
```

When you execute above step, you should see the env name `(taiga)` within parenthesis at the front of the shell identifier `(taiga) [isabell@stardust ~]$`

3.1.2 Test the virutalenv

```shell
(taiga) [isabell@stardust ~]$ python
Python 3.6.8 (default, Apr  2 2020, 13:34:55) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

The python command should take you into python interpreter with the version `3.6.8`. Take note of the version. It should be `3.6.8`. If it is something else, there may be a mistake and you would need to repeat above steps and recontact me.

To get out of the interpreter, type `Ctrl D`



3.1.3 Download `taiga-back` and install requirements.

```shell
(taiga) [isabell@stardust ~]$ git clone https://github.com/taigaio/taiga-back.git taiga-back 
(taiga) [isabell@stardust ~]$ cd taiga-back 
(taiga) [isabell@stardust taiga-back]$ git checkout stable 
(taiga) [isabell@stardust taiga-back]$ pip3.6 install -r requirements.txt
```



3.1.4 Setup `PYTHONPATH`

```shell
(taiga) [isabell@stardust taiga-back]$ export PYTHONPATH=$HOME/.virtualenvs/taiga/lib/python3.6/site-packages
```

3.2 Do database migration

```shell
(taiga) [isabell@stardust taiga-back]$ python manage.py migrate --noinput 
(taiga) [isabell@stardust taiga-back]$ python manage.py loaddata initial_user 
(taiga) [isabell@stardust taiga-back]$ python manage.py loaddata initial_project_templates 
(taiga) [isabell@stardust taiga-back]$ python manage.py collectstatic --noinput
```



3.3 Test if Django installation works

```shell
(taiga) [isabell@stardust taiga-back]$ python manage.py runserver
```

To get out, type `Ctrl C`

#### Notes:

Open the `settings/common.py` file in the taiga-back folder 

```shell
(taiga) [isabell@stardust taiga-back]$ vi settings/common.py
```

and update the following variables. If they are not there please add.

N1. The email configuration for django

For `SMTPHOST`,   `EMAIL_HOST`  and `EMAIL_HOST_PASSWORD` have a look at your uberspace Dashboard. Replace the port number with the one you wrote down earlier. 

```
SMTP_AUTH = True
SMTP_USE_TLS = True
SMTPHOST = 'stardust.uberspace.de'
SMTPPORT = '587'

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
#EMAIL_USE_TLS = True 
#EMAIL_USE_SSL = True # You cannot use both (TLS and SSL) at the same time!
EMAIL_HOST = 'stardust.uberspace.de'
EMAIL_PORT = 587
EMAIL_HOST_USER = 'taiga'
EMAIL_HOST_PASSWORD = 'fsdaspdtr!4R'
```

N2. Databases

```
DATABASES = {
     "default": {
         "ENGINE": "django.db.backends.postgresql",
         "NAME": "taiga",
         "HOST": "localhost",
         "PORT": '63743'
     }
 }
```

N3. Sites

```
 SITES = {
    "api": {"domain": "localhost:63743", "scheme": "http", "name": "api"},
    "front": {"domain": "taiga.uber.space", "scheme": "http", "name": "front"},
 }
```

N4. Secret key

```
SECRET_KEY = "1234567890123456789012345678901234567890123456789012345678901234"
```





### 4 Install taiga-front

4.1 Use ruby version 2.7 available on uberspace

```shell
(taiga) [isabell@stardust taiga-back]$ uberspace tools version list ruby
- 2.5
- 2.6
- 2.7

(taiga) [isabell@stardust taiga-back]$ uberspace tools version show ruby 
Using 'Ruby' version: '2.7'
```

4.2 Install sass and scss_lint

```shell
(taiga) [isabell@stardust taiga-back]$ gem install --user-install sass scss_lint
WARNING:  You don't have /home/isabell/.gem/ruby/2.7.0/bin in your PATH,
	  gem executables will not run.

Ruby Sass has reached end-of-life and should no longer be used.

* If you use Sass as a command-line tool, we recommend using Dart Sass, the new
  primary implementation: https://sass-lang.com/install

* If you use Sass as a plug-in for a Ruby web framework, we recommend using the
  sassc gem: https://github.com/sass/sassc-ruby#readme

* For more details, please refer to the Sass blog:
  https://sass-lang.com/blog/posts/7828841

Successfully installed sass-3.7.4
Successfully installed scss_lint-0.59.0
2 gems installed
```

4.3 Create `~/.npmrc`

```shell
(taiga) [isabell@stardust taiga-back]$ vi ~/.npmrc
```

Insert

```
cat > ~/.npmrc <<__EOF__
prefix=$HOME
umask=077
__EOF__
```

4.4 Install gulp and bower

```shell
(taiga) [isabell@stardust taiga-back]$ npm install -g --prefix=$HOME gulp bower
npm WARN deprecated bower@1.8.8: We don't recommend using Bower for new projects. Please consider Yarn and Webpack or Parcel. You can read how to migrate legacy project here: https://bower.io/blog/2017/how-to-migrate-away-from-bower/
npm WARN deprecated chokidar@2.1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated urix@0.1.0: Please see https://github.com/lydell/urix#deprecated
npm WARN deprecated resolve-url@0.2.1: https://github.com/lydell/resolve-url#deprecated
npm WARN deprecated fsevents@1.2.13: fsevents 1 will break on node v14+ and could be using insecure binaries. Upgrade to fsevents 2.
debug2: channel 0: window 999271 sent adjust 49305ill extract async-done@1.3.2
/home/isabell/bin/bower -> /home/isabell/lib/node_modules/bower/bin/bower
/home/isabell/bin/gulp -> /home/isabell/lib/node_modules/gulp/bin/gulp.js
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@^1.2.7 (node_modules/gulp/node_modules/chokidar/node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.13: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

+ gulp@4.0.2
+ bower@1.8.8
added 332 packages from 222 contributors in 18.452s
```

4.5 Download and extract taiga-front

```shell
(taiga) [isabell@stardust taiga-back]$  mkdir ~/mrr
(taiga) [isabell@stardust taiga-back]$  cd ~/mrr
(taiga) [isabell@stardust  mrr]$ git clone https://github.com/taigaio/taiga-front.git taiga-front
Cloning into 'taiga-front'...
remote: Enumerating objects: 223, done.
remote: Counting objects: 100% (223/223), done.
remote: Compressing objects: 100% (154/154), done.
remote: Total 71380 (delta 113), reused 124 (delta 65), pack-reused 71157
Receiving objects: 100% (71380/71380), 41.55 MiB | 13.24 MiB/s, done.
Resolving deltas: 100% (51098/51098), done.
(taiga) [isabell@stardust  mrr]$  cd taiga-front/
(taiga) [isabell@stardust  taiga-front]$ git checkout stable
Branch 'stable' set up to track remote branch 'stable' from 'origin'.
Switched to a new branch 'stable'
```

4.6 Install node modules

```shell
(taiga) [isabell@stardust taiga-front]$ cd taiga-front
(taiga) [isabell@stardust taiga-front]$ npm install
```

~~4.7 Install bower (it did not work out for me)~~

```shell
(taiga) [isabell@stardust taiga-front]$ bower install
bower ENOENT        No bower.json present
```

4.8 Create taiga-front config

```shell
(taiga) [isabell@stardust taiga-front]$ cp conf/conf.example.json conf/conf.json
```

Edit `~/mrr/taiga-front/conf/conf.json`

```shell
(taiga) [isabell@stardust taiga-front]$  vi ~/mrr/taiga-front/conf/main.json
```

Put the following in 

```
{
    "api": "https://isabell.uber.space/api/v1/",
    "eventsUrl": null,
    "eventsMaxMissedHeartbeats": 5,
    "eventsHeartbeatIntervalTime": 60000,
    "eventsReconnectTryInterval": 10000,
    "debug": true,
    "debugInfo": false,
    "defaultLanguage": "en",
    "themes": ["taiga"],
    "defaultTheme": "taiga",
    "defaultLoginEnabled": true,
    "publicRegisterEnabled": true,
    "feedbackEnabled": true,
    "supportUrl": "https://tree.taiga.io/support/",
    "privacyPolicyUrl": null,
    "termsOfServiceUrl": null,
    "GDPRUrl": null,
    "maxUploadFileSize": null,
    "contribPlugins": [],
    "tagManager": { "accountId": null },
    "tribeHost": null,
    "importers": [],
    "gravatar": false,
    "rtlLanguages": ["fa"]
}
```

4.9 Build the front-end

```shell
(taiga) [isabell@stardust taiga-front]$ gulp deploy
[21:09:43] Using gulpfile ~/mrr/taiga-front/gulpfile.js
[21:09:43] Starting 'deploy'...
[21:09:43] Starting 'clear'...
[21:09:43] Starting 'clear-sass-cache'...
[21:09:43] Finished 'clear-sass-cache' after 747 μs
[21:09:43] Starting '<anonymous>'...
[21:09:43] Finished '<anonymous>' after 1.16 ms
[21:09:43] Finished 'clear' after 3.34 ms
[21:09:43] Starting 'delete-old-version'...
[21:09:43] Finished 'delete-old-version' after 3.66 ms
[21:09:43] Starting 'delete-tmp'...
[21:09:43] Finished 'delete-tmp' after 1.27 ms
[21:09:43] Starting 'copy'...
[21:09:43] Starting 'jade-deploy'...
[21:09:43] Starting 'app-deploy'...
[21:09:43] Starting 'jslibs-deploy'...
[21:09:43] Starting 'link-images'...
[21:09:43] Starting 'compile-themes'...
[21:09:43] Starting 'copy-fonts'...
[21:09:43] Starting 'copy-theme-fonts'...
[21:09:43] Starting 'copy-images'...
[21:09:43] Starting 'copy-emojis'...
[21:09:43] Starting 'copy-prism'...

.
.
.
```

4.10 Copy the frontend files to html folder

Replace `isabell` with your user name

```shell
(taiga) [isabell@stardust taiga-front]$ cd -
(taiga) [isabell@stardust mrr]$ cp -rf taiga-front/dist/* /var/www/virtual/isabell/html/
(taiga) [isabell@stardust mrr]$ cd
(taiga) [isabell@stardust ~]$ cp -rf taiga-back/static /var/www/virtual/isabell/html/
(taiga) [isabell@stardust ~]$ cp -rf taiga-back/media /var/www/virtual/isabell/html/
```

4.11 Create `.htaccess` file, but firtst get your ip.  

```shell
(taiga) [isabell@stardust ~]$ ip a
(taiga) [isabell@stardust ~]$ vi ~/html/.htaccess
```

and put the following content (Have look at your Uberspace Dahsboard for the correct ip) Replace the port number with the one you wrote down earlier.

```
RewriteEngine On

# api requests
RewriteCond %{REQUEST_URI} ^/api
RewriteRule ^(.*)$ http://185.26.156.228:62529/$1 [P]
RequestHeader set X-Forwarded-Proto https env=HTTPS

# deliver files, if not a file deliver index.html
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_URI} !^/static/
RewriteCond %{REQUEST_URI} !^/media/
RewriteRule ^ index.html [L]
```



4.12 Enable logs for debugging

```shell
(taiga) [isabell@stardust ~]$ uberspace web log apache_error enable
```

4.13 Use apache as the web backend

```shell
(taiga) [isabell@stardust ~]$ uberspace web backend set --apache ~/html/
```

4.14 Finally good to run the server and test it in the browser. (Have a look at your Uberspace dashboard for your correct ip) Replace the port number with the one you wrote down earlier

```shell
(taiga) [isabell@stardust ~]$ cd taiga-back
(taiga) [isabell@stardust taiga-back]$ python manage.py runserver 185.26.156.228:63743
```

open the url in browser https://isabell.uber.space



4.15 Run server permanent:

```shell
(taiga) [isabell@stardust taiga-back]$ screen -S taiga-back
(taiga) [isabell@stardust taiga-back]$ source ~/.virtualenvs/taiga/bin/activate
(taiga) [isabell@stardust taiga-back]$ python manage.py runserver 185.26.156.228:63743
```

After this you can safely detach from the screen session by pressing `Ctrl A` and then `d`

Now even if you disconnect from the server, the manage.py keeps on running

whenever you want to attach again to the process, you can do

```shell
[isabell@stardust ~]$ screen -r taiga-back
```



### Good to know



#### Which taiga version do i have installed:

```bash
[isabell@stardust ~]$ cd ../taiga-back
[isabell@stardust taiga-back]$ git describe --tags
```

```shell
[isabell@stardust ~]$ cd ../taiga-front
[isabell@stardust taiga-front]$ git describe --tags
```



#### Exporting / Importing projects

Exporting Project

```shell
(taiga) [isabell@stardust taiga-back]$ python manage.py dump_project PROJECT_SLUG
```

Importing Project:

```shell
(taiga) [isabell@stardust taiga-back]$ python manage.py load_dump file.json owner_email
```



#### Moving taiga from one server to another

Export:

```bash
$ pg_dump taiga > taiga.sql
# tar everything if you want to move to another machine
$ tar -cvzf taigamedia.tar.gz taiga-back/media
$ scp taiga.sql user@destination.domain.com:~
$ scp taigamedia.tar.gz user@destination.domain.com:~ 
```

Import:

```bash
(assume user taiga)
$ cd ~
$ tar -xvzf taigamedia.tar.gz
$ sudo su postgres
$ dropdb taiga && createdb taiga
$ psql taiga < taiga.sql
# now change owner back to taiga
$ psql
postgres=#   alter database taiga owner to taiga;
postgres=#   \q 
exit
$ sudo servicectl restart taiga
```



#### How to upgrade Taiga:

https://taigaio.github.io/taiga-doc/dist/upgrades.html





### References:

1. https://gist.github.com/shuairan/160015f071ef080e841d

2. http://taigaio.github.io/taiga-doc/dist/setup-production.html

3. https://github.com/HepHepHurra/Taiga-on-Uberspace

4. https://github.com/Uberspace/lab/blob/fba013054549ab3f9ebc00e46217f06bc618a41e/source/guide_postgresql.rst

5. https://github.com/taigaio/taiga-doc/issues/133

   

