# Development Environment for Tethys
# CKAN 2.2
# Ubuntu 12.04 LTS
# PostgreSQL 9.3
# PostGIS 2.1
# TethysApps Latest
#
# VERSION 1.0.0

FROM ubuntu:12.04

MAINTAINER Nathan Swain nathan.swain@byu.net

# Apt setup ------------------------------------------------------------------#
RUN apt-get update
RUN apt-get install -y wget sudo curl ssh vim

# Install CKAN dependencies --------------------------------------------------#
RUN apt-get install -y python-dev python-pip python-virtualenv git-core

# Install Production Servers -------------------------------------------------#
RUN apt-get install -y apache2 libapache2-mod-wsgi
RUN apt-get install -y nginx
RUN apt-get install -y postfix

# Drivers for PostgreSQL -----------------------------------------------------#
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list
RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get install -y --force-yes libpq-dev

# Install supervisor ---------------------------------------------------------#
RUN apt-get install -y supervisor

# Home dirs
ENV CKAN_HOME /usr/lib/ckan
ENV TETHYS_DEV_HOME $CKAN_HOME/tethys_dev
ENV APPS_PROJECTS_DEV $CKAN_HOME/apps_projects
ENV CKANEXT $CKAN_HOME/default/src/ckan/ckanext
ENV CKANAPP $CKANEXT/tethys_apps/ckanapp
ENV VENV_HOME /usr/lib/ckan/default
ENV VENV_ACTIVATE /usr/lib/ckan/default/bin/activate

# Configuration files
ENV CKAN_INI /etc/ckan/default/production.ini
ENV TETHYS_INI /usr/lib/ckan/default/src/ckan/ckanext/tethys_apps/tethys_apps.ini

# Create virtual environment -------------------------------------------------#
RUN mkdir -p $VENV_HOME; \
    chown `whoami` $VENV_HOME; \
    virtualenv --no-site-packages /usr/lib/ckan/default

# Install CKAN 2.2 from source -----------------------------------------------#
RUN . $VENV_ACTIVATE && \
    pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.2#egg=ckan' && \
    pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt && \
    deactivate
    
# Make Tethys dev directories ------------------------------------------------#
RUN mkdir -p $TETHYS_DEV_HOME
RUN mkdir -p $APPS_PROJECTS_DEV

# Get Tethys plugin source ---------------------------------------------------#
RUN git clone https://swainn@bitbucket.org/swainn/ckanext-tethys_apps.git $TETHYS_DEV_HOME/ckanext-tethys_apps
RUN git clone https://swainn@bitbucket.org/swainn/ckanext-ciwater_theme.git $TETHYS_DEV_HOME/ckanext-ciwater_theme

# Install plugins into virtual environment -----------------------------------#
RUN . $VENV_ACTIVATE && \
	cd $TETHYS_DEV_HOME/ckanext-tethys_apps && \
    python setup.py install && \
    cd $TETHYS_DEV_HOME/ckanext-ciwater_theme && \
    python setup.py install && \
    deactivate
 
# Symbolically link the plugins ----------------------------------------------#
RUN cp -R $TETHYS_DEV_HOME/ckanext-tethys_apps/ckanext/tethys_apps $CKANEXT/ && \
    cp -R $TETHYS_DEV_HOME/ckanext-ciwater_theme/ckanext/ciwater_theme $CKANEXT/

# Create config and edit -----------------------------------------------------#
RUN mkdir -p /etc/ckan/default && \
    chown -R `whoami` /etc/ckan/ && \
    . $VENV_ACTIVATE && cd $VENV_HOME/src/ckan && \
    paster make-config ckan $CKAN_INI && \
    deactivate

# Add config file ------------------------------------------------------------#
ADD tethys_deploy.conf /tmp/

# CKAN INI
RUN SERVER_ADDRESS=$(grep -Po "(?<=^SERVER_ADDRESS ).*" /tmp/tethys_deploy.conf) && \
    EXTRA_RESOURCE_FIELDS="ckan.extra_resource_fields = model" && \
    PLUGINS="ckan.plugins = stats text_preview recline_preview pdf_preview tethys_apps ciwater_theme" && \
    sed "s/ckan.site_url.*/ckan.site_url = http:\/\/SERVER_ADDRESS/g" -i $CKAN_INI && \
    sed "s/#ckan.storage_path.*/ckan.storage_path = \/var\/lib\/ckan\/default/g" -i $CKAN_INI && \
    sed "s/ckan.plugins =.*/$PLUGINS\n$EXTRA_RESOURCE_FIELDS/g" -i $CKAN_INI

# Link who.ini
RUN ln -sf $CKAN_HOME/default/src/ckan/who.ini /etc/ckan/default/who.ini

# Create filestore directory -------------------------------------------------#
RUN mkdir -p /var/lib/ckan/default

# Create a directory for scripts ---------------------------------------------#
RUN mkdir /usr/lib/ckan/scripts

# Add scripts
ADD scripts/startup.sh /usr/lib/ckan/scripts/
ADD scripts/install_apps.sh /usr/lib/ckan/scripts/

# Set permissions for scripts
RUN chmod -R 755 /usr/lib/ckan/scripts

# Add server scripts
ADD apache/apache.wsgi /etc/ckan/default/
ADD apache/ckan_default /etc/apache2/sites-available/
ADD nginx/ckan_default /etc/nginx/sites-available/

# Apply server name to apache config
RUN SERVER_ADDRESS=$(grep -Po "(?<=^SERVER_ADDRESS ).*" /tmp/tethys_deploy.conf) && \
    sed "s/    ServerName 127.0.0.1/    ServerName $SERVER_ADDRESS/g" -i /etc/apache2/sites-available/ckan_default && \
    sed "s/    ServerAlias 127.0.0.1/    ServerAlias $SERVER_ADDRESS/g" -i /etc/apache2/sites-available/ckan_default

# Add the supervisor conf file to appropriate place --------------------------#
ADD supervisord.conf /etc/supervisor/conf.d/

# Modify listening ports for apache
RUN sed "s/NameVirtualHost \*:80/#NameVirtualHost *:8080/g" -i /etc/apache2/ports.conf && \
    sed "s/Listen 80/Listen 8080/g" -i /etc/apache2/ports.conf && \
    echo "ServerName localhost" | sudo tee /etc/apache2/conf.d/fqdn && \
    echo "*               soft    nofile          8192" >> /etc/security/limits.conf && \
    echo "*               hard    nofile          8192" >> /etc/security/limits.conf && \
    echo "root            soft    nofile          8192" >> /etc/security/limits.conf && \
    echo "root            hard    nofile          8192" >> /etc/security/limits.conf

# Configure apache2 and nginx ------------------------------------------------#
RUN service nginx stop && \
    service apache2 stop

# Enable and disable appropriate sites
RUN a2dissite default && \
    a2ensite ckan_default && \
    ln -s /etc/nginx/sites-available/ckan_default /etc/nginx/sites-enabled/ckan_default

# Handle Apache Permissions
RUN chmod 644 /etc/ckan/default/apache.wsgi

# Change the working directory
WORKDIR /usr/lib/ckan/scripts

EXPOSE 80

CMD ["bash", "/usr/lib/ckan/scripts/startup.sh"]


