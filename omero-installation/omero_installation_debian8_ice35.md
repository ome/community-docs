### OMERO.server installation on Debian 8 and Ice 3.5

This is an example walkthrough for installing OMERO 5.2.x on Debian 8, using a dedicated system use.
These instructions assume your Linux distribution is configured with a UTF-8 locale.

#### Name: OMERO team

#### Institute: University of Dundee

#### Contact: OME External Developer List <ome-devel@lists.openmicroscopy.org.uk>

#### Context:

For convenience in this walkthrough the main OMERO configuration options have been defined as environment variables. When following this walkthrough you can either use your own values:
 
  	OMERO_DB_USER=db_user
	OMERO_DB_PASS=db_password
	OMERO_DB_NAME=omero_database
	OMERO_ROOT_PASS=omero_root_password
	OMERO_DATA_DIR=/OMERO

	OMERO_WEB_PORT=80

	export OMERO_DB_USER OMERO_DB_PASS OMERO_DB_NAME OMERO_ROOT_PASS OMERO_DATA_DIR OMERO_WEB_PORT

	export PGPASSWORD="$OMERO_DB_PASS"

#### System Specifications:

Debian 8 with Java 1.8, Ice 3.5 and PostgreSQL 9.4. Nginx 1.8 for OMERO.web

#### Installation:

The following steps describe how to install an OMERO.server and to deploy OMERO.web
using Nginx.

##### Step 1:

As **root**, Install:

Java 1.8:

	echo 'deb http://httpredir.debian.org/debian jessie-backports main' > /etc/apt/sources.list.d/jessie-backports.list
	apt-get update
	apt-get -y install openjdk-8-jre-headless

Various dependencies:
	
	apt-get -y install \
	python-{matplotlib,numpy,pip,scipy,tables,virtualenv}

	apt-get -y install \
	libtiff5-dev \
	libjpeg62-turbo-dev \
	zlib1g-dev \
	libfreetype6-dev \
	liblcms2-dev \
	libwebp-dev \
	tcl8.6-dev \
	tk8.6-dev

	pip install --upgrade pip

Pillow: 
	
	pip install Pillow<3.0

Ice 3.5:

	apt-get -y install ice-services python-zeroc-ice

Postgres 9.4:

	apt-get -y install apt-transport-https
	add-apt-repository -y "deb https://apt.postgresql.org/pub/repos/apt/ trusty-pgdg main"
	wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
	apt-get update
	apt-get -y install postgresql-9.4
	service postgresql start

##### Step 2:

As **root**, create an omero system user and directory for the OMERO repository

	useradd -m omero
	chmod a+X ~omero

	mkdir -p "$OMERO_DATA_DIR"
	chown omero "$OMERO_DATA_DIR"

##### Step 3:

As **root**, create a database user and a database

	echo "CREATE USER $OMERO_DB_USER PASSWORD '$OMERO_DB_PASS'" | \
    su - postgres -c psql
	su - postgres -c "createdb -E UTF8 -O '$OMERO_DB_USER' '$OMERO_DB_NAME'"

	psql -P pager=off -h localhost -U "$OMERO_DB_USER" -l

##### Step 4:

As the **omero** system user,

Install the lastest 5.2 OMERO.server:

	cd ~omero
	SERVER=http://downloads.openmicroscopy.org/latest/omero5.2/server-ice35.zip
	wget $SERVER -O OMERO.server-ice35.zip
	unzip -q OMERO.server*

Configure the server:

	ln -s OMERO.server-*/ OMERO.server
	OMERO.server/bin/omero config set omero.data.dir "$OMERO_DATA_DIR"
	OMERO.server/bin/omero config set omero.db.name "$OMERO_DB_NAME"
	OMERO.server/bin/omero config set omero.db.user "$OMERO_DB_USER"
	OMERO.server/bin/omero config set omero.db.pass "$OMERO_DB_PASS"
	OMERO.server/bin/omero db script -f OMERO.server/db.sql "" "" "$OMERO_ROOT_PASS"
	OMERO.server/bin/omero db script -f OMERO.server/db.sql --password "$OMERO_ROOT_PASS"
	psql -h localhost -U "$OMERO_DB_USER" "$OMERO_DB_NAME" < OMERO.server/db.sql

##### Step 5:

As **root**, install [Nginx](https://www.nginx.com/resources/wiki/) version 1.8:

	echo "deb http://nginx.org/packages/debian/ jessie nginx" >> /etc/apt/sources.list
	wget http://nginx.org/keys/nginx_signing.key
	apt-key add nginx_signing.key
	rm nginx_signing.key
	apt-get update
	apt-get -y install nginx

As **root**, install the requirements for OMERO.web:

	pip install -r ~omero/OMERO.server/share/web/requirements-py27-nginx.txt

As the **omero** system user,

	configure OMERO.web
	OMERO.server/bin/omero config set omero.web.application_server wsgi-tcp
	OMERO.server/bin/omero web config nginx --http "$OMERO_WEB_PORT" > OMERO.server/nginx.conf.tmp

As **root**, move configuration file and start Nginx:

	mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.disabled
	cp ~omero/OMERO.server/nginx.conf.tmp /etc/nginx/conf.d/omero-web.conf

	service nginx start

### Start OMERO:

You can now start the OMERO.server and OMERO.web manually:

	su - omero -c "OMERO.server/bin/omero admin start"
	su - omero -c "OMERO.server/bin/omero web start"

Few scripts to start automatically are provided, see for example
[linux walkthrough/Running OMERO](https://www.openmicroscopy.org/site/support/omero5.2/sysadmins/unix/server-linux-walkthrough.html)



