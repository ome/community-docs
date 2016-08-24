### OMERO.server installation on CentOS 6 with scl Python 2.7 and Ice 3.5

This is an example walkthrough for installing OMERO 5.2.x on CentOS 6 with scl Python 2.7, using a dedicated system user.
These instructions assume your Linux distribution is configured with a UTF-8 locale.

#### Name: OMERO team

#### Institute: University of Dundee

#### Contact: OME External Developer List <ome-devel@lists.openmicroscopy.org.uk>

#### Context:

For convenience in this walkthrough the main OMERO configuration options have been defined as environment variables. When following this walkthrough you should use your own values:
 
  	OMERO_DB_USER=db_user
	OMERO_DB_PASS=db_password
	OMERO_DB_NAME=omero_database
	OMERO_ROOT_PASS=omero_root_password
	OMERO_DATA_DIR=/OMERO

	OMERO_WEB_PORT=80

	export OMERO_DB_USER OMERO_DB_PASS OMERO_DB_NAME OMERO_ROOT_PASS OMERO_DATA_DIR OMERO_WEB_PORT

	export PGPASSWORD="$OMERO_DB_PASS"

#### System Specifications:

CentOS 6 with scl Python 2.7, Java 1.8, Ice 3.5 and PostgreSQL 9.4. Nginx 1.8 for OMERO.web

#### Installation:

The following steps describe how to install an OMERO.server and to deploy OMERO.web
using Nginx.
We assume that common packages like ``wget``, ``unzip``, ``tar`` are already installed.

##### Step 1:

As **root**, Install:

Various dependencies:
	
	yum -y install epel-release
	yum -y install centos-release-SCL

	yum -y install \
	python27 \
	python27-virtualenv \
	python27-numpy \
	libjpeg-devel \
	libpng-devel \
	libtiff-devel \
	zlib-devel \
	hdf5-devel \
	freetype-devel \
	expat-devel

Java 1.8:

	yum -y install java-1.8.0-openjdk

Others packages required for Ice 3.5:

	yum -y groupinstall "Development Tools"

pip and few requirements for OMERO:

	set +u
	source /opt/rh/python27/enable
	set -u
	easy_install pip

	export PYTHONWARNINGS="ignore:Unverified HTTPS request"

	pip install tables
	pip install matplotlib
	pip install "Pillow<3.0"

Ice 3.5:

	curl -o /etc/yum.repos.d/zeroc-ice-el6.repo \
	http://download.zeroc.com/Ice/3.5/el6/zeroc-ice-el6.repo

	yum -y install db53 db53-utils mcpp

	mkdir /tmp/ice-download
	cd /tmp/ice-download

	wget http://downloads.openmicroscopy.org/ice/experimental/Ice-3.5.1-b1-centos6-sclpy27-x86_64.tar.gz

	tar -zxvf /tmp/ice-download/Ice-3.5.1-b1-centos6-sclpy27-x86_64.tar.gz

	# Install under /opt
	mv Ice-3.5.1-b1-centos6-sclpy27-x86_64 /opt/Ice-3.5.1

	# make path to Ice globally accessible
	# if globally set, there is no need to export LD_LIBRARY_PATH
	echo /opt/Ice-3.5.1/lib64 > /etc/ld.so.conf.d/ice-x86_64.conf
	ldconfig


Postgres 9.4 and reconfigure to allow TCP connections:

	yum -y install postgresql94-server postgresql94

	service postgresql-9.4 initdb
	sed -i.bak -re 's/^(host.*)ident/\1md5/' /var/lib/pgsql/9.4/data/pg_hba.conf
	chkconfig postgresql-9.4 on
	service postgresql-9.4 start

##### Step 2:

As **root**, create an omero system user and directory for the OMERO repository

	useradd -m omero
	chmod a+X ~omero

	mkdir -p "$OMERO_DATA_DIR"
	chown omero "$OMERO_DATA_DIR"

Configure the ``PATH``, ``PYTHONPATH`` etc.
The following lines can, for example, be added to ``.bashrc``

	source /opt/rh/python27/enable
	export ICE_HOME=/opt/Ice-3.5.1
	export PATH="$ICE_HOME/bin:$PATH"
	#Remove commented out export below if Ice is not set globally accessible
	#export LD_LIBRARY_PATH="$ICE_HOME/lib64:$ICE_HOME/lib:$LD_LIBRARY_PATH"
	export PYTHONPATH="$ICE_HOME/python:$PYTHONPATH"
	export SLICEPATH="$ICE_HOME/slice"


##### Step 3:

As **root**, create a database user and a database

	echo "CREATE USER $OMERO_DB_USER PASSWORD '$OMERO_DB_PASS'" | \
    su - postgres -c psql
	su - postgres -c "createdb -E UTF8 -O '$OMERO_DB_USER' '$OMERO_DB_NAME'"

	psql -P pager=off -h localhost -U "$OMERO_DB_USER" -l

##### Step 4:

As the **omero** system user,

Install the latest 5.2 OMERO.server:

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

	set +u
	source /opt/rh/python27/enable
	set -u

	cat << EOF > /etc/yum.repos.d/nginx.repo
	[nginx]
	name=nginx repo
	baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
	gpgcheck=0
	enabled=1
	EOF

	yum -y install nginx

As **root**, install the requirements for OMERO.web:

	pip install -r ~omero/OMERO.server/share/web/requirements-py27-nginx.txt

As the **omero** system user, configure OMERO.web

	OMERO.server/bin/omero config set omero.web.application_server wsgi-tcp
	OMERO.server/bin/omero web config nginx --http "$OMERO_WEB_PORT" > OMERO.server/nginx.conf.tmp

As **root**, move configuration file and start Nginx:

	mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.disabled
	cp ~omero/OMERO.server/nginx.conf.tmp /etc/nginx/conf.d/omero-web.conf

	service nginx reload

### Start OMERO:

You can now start the OMERO.server and OMERO.web manually:

	su - omero -c "OMERO.server/bin/omero admin start"
	su - omero -c "OMERO.server/bin/omero web start"

To start OMERO.server and OMERO.web automatically, please read
[linux walkthrough/Running OMERO](https://www.openmicroscopy.org/site/support/omero5.2/sysadmins/unix/server-linux-walkthrough.html)
