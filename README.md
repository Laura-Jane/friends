# friends

#!/bin/bash
#
# seafile-server-installer/seafile-ubuntu
#
# Copyright 2015, Alexander Jackson <alexander.jackson@seafile.de>
# Copyright 2016, Zheng Xie <xie.zheng@seafile.com>
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU Affero General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Affero General Public License for more details.
#
# You should have received a copy of the GNU Affero General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#
#

if [[ $HOME == "" ]]; then
    export HOME=/root
fi

if [[ $SEAFILE_DEBUG != "" ]]; then
    set -x
fi
set -e

if [[ "$#" -ne 1 ]]; then
    echo "You must specify Seafile version to install"
    echo "Like: $0 7.0.0"
    exit 1
fi

clear

echo "This script will install Seafile Community Edition for you."
echo

# -------------------------------------------
# Vars
# -------------------------------------------
SEAFILE_ADMIN=admin@seafile.local
SEAFILE_SERVER_USER=seafile
SEAFILE_SERVER_HOME=/opt/seafile
IP_OR_DOMAIN=127.0.0.1
SEAFILE_VERSION=$1
TIME_ZONE=Europe/Berlin


SEAFILE_SERVER_PACKAGE=seafile-server_${SEAFILE_VERSION}_x86-64.tar.gz
SEAFILE_SERVER_PACKAGE_URL=https://download.seadrive.org/${SEAFILE_SERVER_PACKAGE}
INSTALLPATH=${SEAFILE_SERVER_HOME}/seafile-server-${SEAFILE_VERSION}/


# -------------------------------------------
# Ensure we are running the installer as root
# -------------------------------------------
if [[ $EUID -ne 0 ]]; then
  echo "  Aborting because you are not root" ; exit 1
fi


# -------------------------------------------
# Abort if directory SEAFILE_SERVER_HOME exists
# -------------------------------------------
if [[ -d "${SEAFILE_SERVER_HOME}" ]] ;
then
  echo "  Aborting because directory ${SEAFILE_SERVER_HOME} already exist" ; exit 1
fi

# -------------------------------------------
# Abort if seafile user exists
# -------------------------------------------
if getent passwd ${SEAFILE_SERVER_USER} > /dev/null 2>&1 ;
then
  echo "Aborting because user ${SEAFILE_SERVER_USER} already exist" ; exit 1
fi


# -------------------------------------------
# Additional requirements
# -------------------------------------------
apt-get update

if [[ ${SEAFILE_VERSION} =~ 7\.1[0-9]*\.[0-9]* ]]; then
    apt-get install -y python3 python3-setuptools python3-pip sudo \
    openjdk-11-jre memcached libmemcached-dev zlib1g-dev pwgen curl openssl poppler-utils libpython2.7 libreoffice \
    libreoffice-script-provider-python ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy nginx

    pip3 install --timeout=3600 Pillow pylibmc captcha jinja2 sqlalchemy \
    django-pylibmc django-simple-captcha python3-ldap
else
    apt-get install -y python2.7 sudo python-pip python-setuptools python-mysqldb python-ldap python-urllib3 \
    openjdk-11-jre memcached libmemcached-dev zlib1g-dev pwgen curl openssl poppler-utils libpython2.7 libreoffice \
    libreoffice-script-provider-python ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy nginx python-requests

    pip install pylibmc==1.6.0 django-pylibmc==0.6.1
    pip install --timeout=3600 Pillow==4.3.0
    pip install psd-tools==1.4
fi

# -------------------------------------------
# Setup Nginx
# -------------------------------------------

rm /etc/nginx/sites-enabled/*

cat > /etc/nginx/sites-available/seafile.conf << EOF
log_format seafileformat '\$http_x_forwarded_for \$remote_addr [\$time_local] "\$request" \$status \$body_bytes_sent "\$http_referer" "\$http_user_agent" \$upstream_response_time';

server {
    listen 80;
    server_name seafile.example.com;

    proxy_set_header X-Forwarded-For \$remote_addr;

    location / {
         proxy_pass         http://127.0.0.1:8000;
         proxy_set_header   Host \$host;
         proxy_set_header   X-Real-IP \$remote_addr;
         proxy_set_header   X-Forwarded-For \$proxy_add_x_forwarded_for;
         proxy_set_header   X-Forwarded-Host \$server_name;
         proxy_set_header   X-Forwarded-Proto \$scheme;
         proxy_read_timeout  1200s;

         # used for view/edit office file via Office Online Server
         client_max_body_size 0;

         access_log      /var/log/nginx/seahub.access.log seafileformat;
         error_log       /var/log/nginx/seahub.error.log;
    }
    
    location /seafhttp {
         rewrite ^/seafhttp(.*)$ \$1 break;
         proxy_pass http://127.0.0.1:8082;
         client_max_body_size 0;
         proxy_set_header   X-Forwarded-For \$proxy_add_x_forwarded_for;
         proxy_connect_timeout  36000s;
         proxy_read_timeout  36000s;
         proxy_request_buffering    off;

        access_log      /var/log/nginx/seafhttp.access.log seafileformat;
        error_log       /var/log/nginx/seafhttp.error.log;
    }
    location /media {
        root ${SEAFILE_SERVER_HOME}/seafile-server-latest/seahub;
    }
    location /seafdav {
        fastcgi_pass    127.0.0.1:8080;
        fastcgi_param   SCRIPT_FILENAME     \$document_root\$fastcgi_script_name;
        fastcgi_param   PATH_INFO           \$fastcgi_script_name;
        fastcgi_param   SERVER_PROTOCOL     \$server_protocol;
        fastcgi_param   QUERY_STRING        \$query_string;
        fastcgi_param   REQUEST_METHOD      \$request_method;
        fastcgi_param   CONTENT_TYPE        \$content_type;
        fastcgi_param   CONTENT_LENGTH      \$content_length;
        fastcgi_param   SERVER_ADDR         \$server_addr;
        fastcgi_param   SERVER_PORT         \$server_port;
        fastcgi_param   SERVER_NAME         \$server_name;
        fastcgi_param   REMOTE_ADDR         \$remote_addr;

        client_max_body_size 0;

        access_log      /var/log/nginx/seafdav.access.log seafileformat;
        error_log       /var/log/nginx/seafdav.error.log;
    }
}
EOF

ln -sf /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf

service nginx restart


# -------------------------------------------
# MariaDB
# -------------------------------------------
if [[ -f "/root/.my.cnf" ]] ;
then
  echo "MariaDB installed before, skip this part"
  SQLROOTPW=`sed -n 's/password=//p' /root/.my.cnf`
else
  DEBIAN_FRONTEND=noninteractive apt-get install -y mariadb-server

  SQLROOTPW=$(pwgen)

  mysqladmin -u root password $SQLROOTPW

  cat > /root/.my.cnf <<EOF
[client]
user=root
password=$SQLROOTPW
EOF

  chmod 600 /root/.my.cnf
fi

# -------------------------------------------
# Seafile init script // Seafile Server Autostart + Scripte
# -------------------------------------------
cat > /etc/systemd/system/seafile.service <<'EOF'
[Unit]
Description=Seafile Server
After=network.target mysql.service

[Service]
Type=oneshot
ExecStart=/opt/seafile/seafile-server-latest/seafile.sh start
ExecStop=/opt/seafile/seafile-server-latest/seafile.sh stop
RemainAfterExit=yes
User=seafile
Group=nogroup

[Install]
WantedBy=multi-user.target
EOF
systemctl enable seafile
cat > /etc/systemd/system/seahub.service <<'EOF'
[Unit]
Description=Seafile Seahub
After=network.target seafile.service

[Service]
ExecStart=/opt/seafile/seafile-server-latest/seahub.sh start
ExecStop=/opt/seafile/seafile-server-latest/seahub.sh stop
User=seafile
Group=nogroup
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable seahub

# Seafile restart script
cat > /usr/local/sbin/seafile-server-restart << 'EOF'
#!/bin/bash
for ACTION in stop start ; do
    for SERVICE in seafile seahub ; do
      systemctl ${ACTION} ${SERVICE}
    done
done
EOF

chmod 700 /usr/local/sbin/seafile-server-restart

# Seafile start script
cat > /usr/local/sbin/seafile-server-start << 'EOF'
#!/bin/bash
for ACTION in start ; do
    for SERVICE in seafile seahub ; do
      systemctl ${ACTION} ${SERVICE}
    done
done
EOF

chmod 700 /usr/local/sbin/seafile-server-start

# Seafile stop script
cat > /usr/local/sbin/seafile-server-stop << 'EOF'
#!/bin/bash
for ACTION in stop ; do
    for SERVICE in seafile seahub ; do
      systemctl ${ACTION} ${SERVICE}
    done
done
EOF

chmod 700 /usr/local/sbin/seafile-server-stop


# -------------------------------------------
# Seafile
# -------------------------------------------
mkdir -p ${SEAFILE_SERVER_HOME}/installed
cd ${SEAFILE_SERVER_HOME}
if [[ ! -e /opt/${SEAFILE_SERVER_PACKAGE} ]]; then
    curl -OL ${SEAFILE_SERVER_PACKAGE_URL}
else
    cp /opt/${SEAFILE_SERVER_PACKAGE} .
fi
tar xzf ${SEAFILE_SERVER_PACKAGE}

mv ${SEAFILE_SERVER_PACKAGE} installed


# -------------------------------------------
# Seafile DB
# -------------------------------------------
if [[ -f "/opt/seafile.my.cnf" ]] ;
then
  echo "MariaDB installed before, skip this part"
  SQLSEAFILEPW=`sed -n 's/password=//p' /opt/seafile.my.cnf`
else
  SQLSEAFILEPW=$(pwgen)

  cat > /opt/seafile.my.cnf <<EOF
[client]
user=seafile
password=$SQLSEAFILEPW
EOF

  chmod 600 /opt/seafile.my.cnf
fi

# -------------------------------------------
# Add seafile user
# -------------------------------------------
useradd --system --comment "${SEAFILE_SERVER_USER}" ${SEAFILE_SERVER_USER} --home-dir  ${SEAFILE_SERVER_HOME}

# -------------------------------------------
# Go to /opt/seafile/seafile-pro-server-${SEAFILE_VERSION}
# -------------------------------------------
cd $INSTALLPATH

# -------------------------------------------
# Vars - Don't touch these unless you really know what you are doing!
# -------------------------------------------
TOPDIR=$(dirname "${INSTALLPATH}")
DEFAULT_CONF_DIR=${TOPDIR}/conf
SEAFILE_DATA_DIR=${TOPDIR}/seafile-data
DEST_SETTINGS_PY=${TOPDIR}/conf/seahub_settings.py

mkdir -p ${DEFAULT_CONF_DIR}

# -------------------------------------------
# Create ccnet, seafile, seahub conf using setup script
# -------------------------------------------

./setup-seafile-mysql.sh auto -u seafile -w ${SQLSEAFILEPW} -r ${SQLROOTPW}

# -------------------------------------------
# Configure Seafile WebDAV Server(SeafDAV)
# -------------------------------------------
sed -i 's/enabled = .*/enabled = true/' ${DEFAULT_CONF_DIR}/seafdav.conf
sed -i 's/fastcgi = .*/fastcgi = true/' ${DEFAULT_CONF_DIR}/seafdav.conf
sed -i 's/share_name = .*/share_name = \/seafdav/' ${DEFAULT_CONF_DIR}/seafdav.conf

# -------------------------------------------
# Configuring seahub_settings.py
# -------------------------------------------
cat >> ${DEST_SETTINGS_PY} <<EOF

CACHES = {
    'default': {
        'BACKEND': 'django_pylibmc.memcached.PyLibMCCache',
        'LOCATION': '127.0.0.1:11211',
    },
    'locmem': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    },
}
COMPRESS_CACHE_BACKEND = 'locmem'

# EMAIL_USE_TLS                       = False
# EMAIL_HOST                          = 'localhost'
# EMAIL_HOST_USER                     = ''
# EMAIL_HOST_PASSWORD                 = ''
# EMAIL_PORT                          = '25'
# DEFAULT_FROM_EMAIL                  = EMAIL_HOST_USER
# SERVER_EMAIL                        = EMAIL_HOST_USER

TIME_ZONE                           = '${TIME_ZONE}'
SITE_BASE                           = 'http://${IP_OR_DOMAIN}'
SITE_NAME                           = 'Seafile Server'
SITE_TITLE                          = 'Seafile Server'
SITE_ROOT                           = '/'
ENABLE_SIGNUP                       = False
ACTIVATE_AFTER_REGISTRATION         = False
SEND_EMAIL_ON_ADDING_SYSTEM_MEMBER  = True
SEND_EMAIL_ON_RESETTING_USER_PASSWD = True
CLOUD_MODE                          = False
FILE_PREVIEW_MAX_SIZE               = 30 * 1024 * 1024
SESSION_COOKIE_AGE                  = 60 * 60 * 24 * 7 * 2
SESSION_SAVE_EVERY_REQUEST          = False
SESSION_EXPIRE_AT_BROWSER_CLOSE     = False

FILE_SERVER_ROOT                    = 'http://${IP_OR_DOMAIN}/seafhttp'
EOF


# -------------------------------------------
# Backup check_init_admin.py befor applying changes
# -------------------------------------------
cp ${INSTALLPATH}/check_init_admin.py ${INSTALLPATH}/check_init_admin.py.backup


# -------------------------------------------
# Set admin credentials in check_init_admin.py
# -------------------------------------------
SEAFILE_ADMIN_PW=$(pwgen)
eval "sed -i 's/= ask_admin_email()/= \"${SEAFILE_ADMIN}\"/' ${INSTALLPATH}/check_init_admin.py"
eval "sed -i 's/= ask_admin_password()/= \"${SEAFILE_ADMIN_PW}\"/' ${INSTALLPATH}/check_init_admin.py"

# -------------------------------------------
# Start and stop Seafile eco system. This generates the initial admin user.
# -------------------------------------------
${INSTALLPATH}/seafile.sh start
${INSTALLPATH}/seahub.sh start
sleep 2                         # sleep for a while, otherwise seahub will not be stopped
${INSTALLPATH}/seahub.sh stop
sleep 1
${INSTALLPATH}/seafile.sh stop


# -------------------------------------------
# Restore original check_init_admin.py
# -------------------------------------------
mv ${INSTALLPATH}/check_init_admin.py.backup ${INSTALLPATH}/check_init_admin.py

# -------------------------------------------
# Fix permissions
# -------------------------------------------
chown ${SEAFILE_SERVER_USER}:${SEAFILE_SERVER_USER} -R ${SEAFILE_SERVER_HOME}
if [[ -d /tmp/seafile-office-output/ ]]; then
    chown ${SEAFILE_SERVER_USER}:${SEAFILE_SERVER_USER} -R /tmp/seafile-office-output/
fi


# -------------------------------------------
# Database Backup
# -------------------------------------------
# install package
apt-get install automysqlbackup -y

# change destination directory
SEAFILE_SERVER_BACKUPDIR="/var/backups/seafile-server/database/"
eval "sed -i 's#/var/lib/automysqlbackup#$SEAFILE_SERVER_BACKUPDIR#' /etc/default/automysqlbackup"

# launch a backup now
/usr/sbin/automysqlbackup

echo "Automatic database backups are in ${SEAFILE_SERVER_BACKUPDIR}."

# -------------------------------------------
# Start seafile server
# -------------------------------------------
echo "Starting productive Seafile server"
service seafile start
service seahub start


# -------------------------------------------
# Install GParted
# -------------------------------------------
echo "Installing GParted"
apt-get install gparted -y


# -------------------------------------------
# Final report
# -------------------------------------------
cat > ${TOPDIR}/aio_seafile-server.log<<EOF

  Your Seafile server is installed
  -----------------------------------------------------------------

  Server Address:      http://${IP_OR_DOMAIN}

  Seafile Admin:       ${SEAFILE_ADMIN}
  Admin Password:      ${SEAFILE_ADMIN_PW}

  Seafile Data Dir:    ${SEAFILE_DATA_DIR}

  Seafile DB Credentials:  Check /opt/seafile.my.cnf
  Root DB Credentials:     Check /root/.my.cnf

  This report is also saved to ${TOPDIR}/aio_seafile-server.log



  Next you should manually complete the following steps
  -----------------------------------------------------------------

  1) Log in to Seafile and configure your server domain via the system
     admin area if applicable.

  2) If this server is behind a firewall, you need to ensure that
     tcp port 80 is open.

  3) Seahub tries to send emails via the local server. Install and
     configure Postfix for this to work or
     check https://manual.seafile.com/config/sending_email.html
     for instructions on how to use an existing email account via SMTP.




  Optional steps
  -----------------------------------------------------------------

  1) Check seahub_settings.py and customize it to fit your needs. Consult
     http://manual.seafile.com/config/seahub_settings_py.html for possible switches.

  2) Setup NGINX with official SSL certificate, we suggest you use Let’s Encrypt. Check
     https://manual.seafile.com/deploy/https_with_nginx.html

  3) Secure server with iptables based firewall. For instance: UFW or shorewall

  4) Harden system with port knocking, fail2ban, etc.

  5) Enable unattended installation of security updates. Check
     https://wiki.Ubuntu.org/UnattendedUpgrades for details.

  6) Implement a backup routine for your Seafile server.

  7) Update NGINX worker processes to reflect the number of CPU cores.




  Seafile support options
  -----------------------------------------------------------------

  For free community support visit:   https://forum.seafile.com
  For paid commercial support visit:  https://seafile.com

EOF

chmod 600 ${TOPDIR}/aio_seafile-server.log

clear

cat ${TOPDIR}/aio_seafile-server.log
