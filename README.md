ubuntu-statsd-graphite-setup
============================

* Ubuntu 12.04
 * Name: 099720109477/ubuntu/images/ebs/ubuntu-precise-12.04-i386-server-20120822
 * AMI ID: ami-057bcf6c
* Graphite 0.9.10
 * https://launchpad.net/graphite/0.9/0.9.10
* Statsd
 * https://github.com/etsy/statsd

1. Adapted from: http://kaivanov.blogspot.com/2012/02/how-to-install-and-use-graphite.html

```sh
apt-get update
apt-get upgrade
wget http://launchpad.net/graphite/0.9/0.9.10/+download/graphite-web-0.9.10.tar.gz
wget http://launchpad.net/graphite/0.9/0.9.10/+download/carbon-0.9.10.tar.gz
wget http://launchpad.net/graphite/0.9/0.9.10/+download/whisper-0.9.10.tar.gz
tar -zxvf graphite-web-0.9.10.tar.gz
tar -zxvf carbon-0.9.10.tar.gz
tar -zxvf whisper-0.9.10.tar.gz
mv graphite-web-0.9.10 graphite
mv carbon-0.9.10 carbon
mv whisper-0.9.10 whisper
rm carbon-0.9.10.tar.gz
rm graphite-web-0.9.10.tar.gz
rm whisper-0.9.10.tar.gz
apt-get install --assume-yes apache2 apache2-mpm-worker apache2-utils apache2.2-bin apache2.2-common libapr1 libaprutil1 libaprutil1-dbd-sqlite3 libapache2-mod-wsgi libaprutil1-ldap memcached python-cairo python-cairo-dev python-django python-ldap python-memcache python-pysqlite2 sqlite3 erlang-os-mon erlang-snmp rabbitmq-server bzr expect ssh libapache2-mod-python python-setuptools
apt-get install build-essential python-dev
easy_install zope.interface
easy_install twisted
easy_install txamqp
easy_install django-tagging

pushd whisper
 python setup.py install
popd

pushd carbon
 python setup.py install
popd

pushd graphite/conf
 cp carbon.conf.example carbon.conf
 cp storage-schemas.conf.example storage-schemas.conf
popd

pushd graphite
 python check-dependencies.py
 python setup.py install

 pushd examples
  cp example-graphite-vhost.conf /etc/apache2/sites-available/default
  cp /opt/graphite/conf/graphite.wsgi.example /opt/graphite/conf/graphite.wsgi
  vim /etc/apache2/sites-available/default
 popd
popd

```

```
# This needs to be in your server's config somewhere, probably
# the main httpd.conf
# NameVirtualHost *:80

# This line also needs to be in your server's config.
# LoadModule wsgi_module modules/mod_wsgi.so

# You need to manually edit this file to fit your needs.
# This configuration assumes the default installation prefix
# of /opt/graphite/, if you installed graphite somewhere else
# you will need to change all the occurances of /opt/graphite/
# in this file to your chosen install location.

<IfModule !wsgi_module.c>
    LoadModule wsgi_module modules/mod_wsgi.so
</IfModule>

# XXX You need to set this up!
# Read http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGISocketPrefix
#WSGISocketPrefix run/wsgi
WSGISocketPrefix /etc/httpd/wsgi/

<VirtualHost *:80>
        ServerName graphite
        DocumentRoot "/opt/graphite/webapp"
        ErrorLog /opt/graphite/storage/log/webapp/error.log
        CustomLog /opt/graphite/storage/log/webapp/access.log common

        # I've found that an equal number of processes & threads tends
        # to show the best performance for Graphite (ymmv).
        WSGIDaemonProcess graphite processes=5 threads=5 display-name='%{GROUP}' inactivity-timeout=120
        WSGIProcessGroup graphite
        WSGIApplicationGroup %{GLOBAL}
        WSGIImportScript /opt/graphite/conf/graphite.wsgi process-group=graphite application-group=%{GLOBAL}

        # XXX You will need to create this file! There is a graphite.wsgi.example
        # file in this directory that you can safely use, just copy it to graphite.wgsi
        WSGIScriptAlias / /opt/graphite/conf/graphite.wsgi

        Alias /content/ /opt/graphite/webapp/content/
        <Location "/content/">
                SetHandler None
        </Location>

        # XXX In order for the django admin site media to work you
        # must change @DJANGO_ROOT@ to be the path to your django
        # installation, which is probably something like:
        # /usr/lib/python2.6/site-packages/django
        Alias /media/ "@DJANGO_ROOT@/contrib/admin/media/"
        <Location "/media/">
                SetHandler None
        </Location>

        # The graphite.wsgi file has to be accessible by apache. It won't
        # be visible to clients because of the DocumentRoot though.
        <Directory /opt/graphite/conf/>
                Order deny,allow
                Allow from all
        </Directory>
</VirtualHost>
```

2. Adapted from: http://www.kinvey.com/blog/item/158-how-to-set-up-metric-collection-using-graphite-and-statsd-on-ubuntu-1204-lts

```
sudo apt-get install python-software-properties
sudo apt-add-repository ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs npm
sudo apt-get install git

cd /opt && sudo git clone git://github.com/etsy/statsd.git

cat >> /tmp/localConfig.js << EOF
{
  graphitePort: 2003
, graphiteHost: "127.0.0.1"
, port: 8125
}
EOF

sudo cp /tmp/localConfig.js /opt/statsd/localConfig.js
```

References
* http://stackoverflow.com/questions/7099197/tracking-metrics-using-statsd-via-etsy-and-graphite-graphite-graph-doesnt-se
* http://www.kinvey.com/blog/item/158-how-to-set-up-metric-collection-using-graphite-and-statsd-on-ubuntu-1204-lts
* http://www.kinvey.com/blog/item/178-graphite-on-ubuntu-1204-lts-part-ii-gunicorn-nginx-and-supervisord
* http://supervisord.org/
* http://supervisord.org/installing.html#installing-to-a-system-with-internet-access
* http://kaivanov.blogspot.com/2012/02/how-to-install-and-use-graphite.html
