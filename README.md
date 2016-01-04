# Linux Web Server


# Accessing the Server
[IP Address](http://52.33.24.184) - `52.33.24.184` 
[Web Address](http://ec2-52-33-24-184.us-west-2.compute.amazonaws.com) - `ec2-52-33-24-184.us-west-2.compute.amazonaws.com`

SSH with:

```sh
ssh -i ~/.ssh/grader_key -p 2200 grader@52.33.24.184
```
In case you ever need it, `grader` has password `password`, but password login is disabled.

# Configuration Steps

## Step 1: Updating and Upgrading

```sh
sudo apt-get update
sudo apt-get upgrade
sudo apt-get autoremove
```
## Step 2: Changing User Settings and Creating New Ones

```sh
sudo adduser grader
sudo vim /etc/sudoers.d/grader
```
Edited `grader` to make it a sudoer:

```
grader ALL=(ALL) NOPASSWD:ALL
```
Also, add the following line to `/etc/hosts`:

```
127.0.1.1 ip-10-20-23-147
```

## Step 3: Setting up RSA login for `grader`

On local machine:

```sh
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/username/.ssh/id_rsa): grader_key 
...
```
Now, ssh as `grader` and do the following:

```sh
ssh grader@52.33.24.184
# type in password, which in this case is just 'password'
mkdir .ssh
sudo vim .ssh/authorized_keys
```
Now copy and paste content of `grader_key.pub` on your local machine into `authorized_keys` on the server.

Now change the permissions of the folder and file:

```sh
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

## Step 4: Disabling Password Login
Change the `sshd_config` file in the following way:

```sh
sudo vim /etc/ssh/sshd_config
```

```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication yes
```
to

```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```
Now, restart the `ssh` service:

```
sudo service ssh restart
```

## Step 5: Setting up Firewall and Changing Ports
Change the `ssh` port:

```sh
sudo vim /etc/ssh/sshd_config
```
And change

```
# What ports, IPs and protocols we listen for
Port 22
```
to

```
# What ports, IPs and protocols we listen for
Port 2200
```
Restart `ssh`:

```sh
sudo service ssh restart
```

Now, change `ufw` settings and start it:

```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow ntp
sudo ufw allow http
sudo ufw enable
```

## Step 6: Changing Timezone
**Sources**

* [AskUbuntu post](http://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt)

Run the following and just select `Other` into `UTC`.

```
dpkg-reconfiguration tzdata
```


## Step 7: Setting up Packages
From my catalog application's readme:

```sh
sudo apt-get -qqy update
sudo apt-get -qqy install postgresql python-psycopg2
sudo apt-get -qqy install python-flask python-sqlalchemy
sudo apt-get -qqy install python-pip
sudo apt-get -qqy install git
sudo pip install bleach
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
```
Install Apache:

```sh
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
```
Install PostgreSQL:

```sh
sudo apt-get install postgresql
```

## Step 8: Configuring PostgreSQL
**Sources**

* [Postgres Documentation on adding users](http://www.postgresql.org/docs/9.1/static/app-createuser.html)
* [DigitalOcean post on setting up PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-9-4-on-debian-8)

```sh
su - postgres
createuser --interactive -P
```
Set up a user called `catalog` and say no to every option.

When it prompts for a password, I used `8c33481e-9a13-4f23-89bd-8e81beecdd5d`.

Now, create the `catalog` database (as user `postgres`):

```sh
createdb catalog
```

The connect string for this database is `postgres://catalog:8c33481e-9a13-4f23-89bd-8e81beecdd5d@localhost/catalog`.

## Step 9: Setting up Application with Apache
**Sources**

* [Flask documentation for deploying to Apache](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/#creating-a-wsgi-file)

### Part 1 - Make Apache Serve the App
Create the directory structure:

```sh
sudo mkdir /var/www/catalog
sudo git clone https://github.com/Nasafato/LearningOAuthAndFlask.git /var/www/catalog/catalog
```
**NOTE: I made a branch called `web_server` that has all correct configurations for the Flask app itself, so use that if you want to skip all the edits that need to be made to the Flask app**

Now make a whole bunch of changes.

In `/var/www/catalog/catalog`, make a file called `__init__.py` and paste the following into it:

```py
from application import app
if __name__ == "__main__":
	app.run()
```

In `/var/www/catalog`, make a file called `catalog.wsgi` and paste the following into it:

```py
import sys
sys.path.append('/var/www/catalog/catalog')
from catalog import app as application
```
Now, edit `/etc/apache2/sites-enabled/000-default.conf`:

```sh
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/catalog

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
        #WSGIScriptAlias / /var/www/html/myapp.wsgi
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```
### Part 2 - Make the App Work
In `/var/www/catalog/catalog/`, edit `application.py`, `database_setup.py`, and `insert_items.py`.

Replace this line:

```py
engine = create_engine('sqlite:///catalog.db')
```
with this line:

```py
engine = create_engine('postgres://catalog:8c33481e-9a13-4f23-89bd-8e81beecdd5d@localhost/catalog')
```

Change `UPLOAD_FOLDER` to:

```py
UPLOAD_FOLDER = '/var/www/catalog/catalog/static/img/'
```

In `application.py`, change all four instances of `client_secrets.json` or `fb_client_secrets.json` to `/var/www/catalog/catalog/*.json`.

```py
CLIENT_ID = json.loads(
    open('/var/www/catalog/catalog/client_secrets.json',
             'r').read())['web']['client_id']

...

app_id = json.loads(open(
        '/var/www/catalog/catalog/fb_client_secrets.json', 'r').read())[
            'web']['app_id']
app_secret = json.loads(
    open('/var/www/catalog/catalog/fb_client_secrets.json',
         'r').read())['web']['app_secret']

...

oauth_flow = flow_from_clientsecrets(
            '/var/www/catalog/catalog/client_secrets.json', scope='')
```

Change `add_image()` and `delete_image()` to the following:

```py
def add_image(image_file, object_type):
    """Adds an image to the file system of the appropriate type"""
    if (image_file and
            is_allowed_filename(image_file.filename) and
            is_valid_type(object_type)):
        extension = extract_file_extension(image_file.filename)

        # generate unique filename
        filename = "{}.{}".format(str(uuid.uuid4()), extension)
        filename = secure_filename(filename)

        relative_path = os.path.join('/static/img', object_type, filename)
        full_path = os.path.join(app.config['UPLOAD_FOLDER'],
                                 object_type,
                                 filename)
        image_file.save(full_path)
        return relative_path
    else:
        return None


def delete_image(filepath):
    """Removes an image from the filesystem"""
    delete_path = os.path.join('/var/www/catalog/catalog', filepath[1:])
    os.remove(delete_path)
```

Now you can restart Apache:

```sh
sudo service apache2 restart
```
Also, setup the database:

```sh
python database_setup.py
python insert_items.py
```
Now you should be able to see the application when you go to the website address.

All that's left to do is to add the appropriate URLs to the Google Developer Console and to Facebook so that the app lets the user log in to his/her account.

For Google, in **Authorized Javascript origins**, add:

* http://ec2-52-33-24-184.us-west-2.compute.amazonaws.com
* http://52.33.24.184

In **Authorized Redirect URIs**, add:

* http://ec2-52-33-24-184.us-west-2.compute.amazonaws.com/login
* http://ec2-52-33-24-184.us-west-2.compute.amazonaws.com/gconnect

For Facebook, in **Settings** -> **Advanced** -> **Valid OAuth redirect URIs**, add:

* http://ec2-52-33-24-184.us-west-2.compute.amazonaws.com
* http://52.33.24.184

Now the app should work!

## Step 10 - glances and fail2ban
**Sources**

* [Configuring fail2ban](https://blog.vigilcode.com/2011/05/ufw-with-fail2ban-quick-secure-setup-part-ii/)

### Part 1 - Installing `glances`
I first installed `glances`, which is very easy:

```sh
sudo apt-get install glances
```

### Part 2 - Installing and Configuring `fail2ban` with `ufw`
First install it, then start editing `/etc/fail2ban/jail.local`:

```
sudo apt-get install fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Make sure that the individual jail sections look something like this:

```
[ssh]

enabled  = true
banaction = ufw-ssh
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6

[apache]

enabled  = true
banaction = ufw-apache
port     = http,https
filter   = apache-auth
logpath  = /var/log/apache*/*error.log
maxretry = 6

[apache-noscript]

enabled  = true
banaction = ufw-apache
port     = http,https
filter   = apache-noscript
logpath  = /var/log/apache*/*error.log
maxretry = 6

[apache-overflows]

enabled  = true
banaction = ufw-apache
port     = http,https
filter   = apache-overflows
logpath  = /var/log/apache*/*error.log
maxretry = 2
```

Now, set up the `banaction` rules:

Create and edit `/etc/fail2ban/action.d/ufw-ssh.conf`:

```
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = ufw insert 1 deny from <ip> to any app OpenSSH
actionunban = ufw delete deny from <ip> to any app OpenSSH
```

and `/etc/fail2ban/action.d/ufw-apache.conf`:

```
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = ufw insert 2 deny from <ip> to any app "Apache Full"
actionunban = ufw delete deny from <ip> to any app "Apache Full"
```

Now restart:

```
sudo service fail2ban restart
```