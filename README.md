# This manual-install method builds a FreePBX system with the following specifications:
1. FreePBX 17
2. Asterisk  21
3. PHP 8.2
4. Maria DB (v10.11)
5. Node JS (v18.16)
6. ARM64 platform

## Step by step
Start from a base Debian 12 installation. All necessary packages will be installed through the following commands.
```markdown
timedatectl set-timezone Asia/Bangkok
apt-get update
apt-get upgrade
apt-get -y install build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev uuid-dev default-libmysqlclient-dev htop sngrep lame ffmpeg mpg123
apt-get -y install git vim curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev build-essential libjansson-dev libxml2-dev uuid-dev expect cron libedit*
```
## PHP 8.2 Installation
First, download the GPG key.
```markdown
apt-get -y install lsb-release apt-transport-https ca-certificates 
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
```
Next, add SURY to the repository.
```markdown
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/php.list
apt-get update
apt-get upgrade -y && reboot
apt-get -y install php8.2
```
```markdown
apt-get install -y build-essential linux-headers-`uname -r` openssh-server apache2 mariadb-server mariadb-client bison flex php8.2 php8.2-curl php8.2-cli php8.2-common php8.2-mysql php8.2-gd php8.2-mbstring  php8.2-intl php8.2-xml php-pear curl sox libncurses5-dev libssl-dev mpg123 libxml2-dev libnewt-dev sqlite3  libsqlite3-dev pkg-config automake libtool autoconf git unixodbc-dev uuid uuid-dev libasound2-dev libogg-dev libvorbis-dev libicu-dev libcurl4-openssl-dev odbc-mariadb libical-dev libneon27-dev libsrtp2-dev  libspandsp-dev sudo subversion libtool-bin python-dev-is-python3 unixodbc vim wget libjansson-dev software-properties-common nodejs npm ipset iptables fail2ban php-soap
```
## Asterisk Installation
Download Asterisk source and compile
```markdown
cd /usr/src
wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-21-current.tar.gz
tar xvf asterisk-21-current.tar.gz
cd asterisk-21*/
contrib/scripts/get_mp3_source.sh
contrib/scripts/install_prereq install
./configure  --libdir=/usr/lib64 --with-pjproject-bundled --with-jansson-bundled
make menuselect
make
make install
make samples
make config
ldconfig
```
Create an Asterisk user and give permission
```markdown
groupadd asterisk
useradd -r -d /var/lib/asterisk -g asterisk asterisk
usermod -aG audio,dialout asterisk
chown -R asterisk:asterisk /etc/asterisk
chown -R asterisk:asterisk /var/{lib,log,spool}/asterisk
chown -R asterisk:asterisk /usr/lib64/asterisk
   
sed -i 's|#AST_USER|AST_USER|' /etc/default/asterisk
sed -i 's|#AST_GROUP|AST_GROUP|' /etc/default/asterisk
sed -i 's|;runuser|runuser|' /etc/asterisk/asterisk.conf
sed -i 's|;rungroup|rungroup|' /etc/asterisk/asterisk.conf
echo "/usr/lib64" >> /etc/ld.so.conf.d/x86_64-linux-gnu.conf
ldconfig
```
Configure Apache web server
```markdown
sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/8.2/apache2/php.ini
sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
sed -i 's/^\(User\|Group\).*/\1 asterisk/' /etc/apache2/apache2.conf
sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
a2enmod rewrite
systemctl restart apache2
rm /var/www/html/index.html
```
## Install and Configure ODBC
Installing Build Dependencies on Linux
```markdown
apt-get install git cmake make gcc libssl-dev unixodbc odbcinst unixodbc-dev
```
Building MariaDB Connector/ODBC from the Git Repository
```markdown
git clone https://github.com/MariaDB/mariadb-connector-odbc.git
cd mariadb-connector-odbc
```
Building MariaDB Connector/ODBC
```markdown
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCONC_WITH_UNIT_TESTS=Off -DCONC_WITH_MSI=OFF -DCMAKE_INSTALL_PREFIX=/usr/local .
cmake --build . --config RelWithDebInfo
```
To install it, execute the following:
```markdown
make install
```
### Configure ODBC
```markdown
cat <<EOF > /etc/odbcinst.ini
[MySQL]
Description = ODBC for MySQL (MariaDB)
Driver = /usr/local/lib/mariadb/libmaodbc.so
FileUsage = 1
EOF
```
```markdown
cat <<EOF > /etc/odbc.ini
[MySQL-asteriskcdrdb]
Description = MySQL connection to 'asteriskcdrdb' database
Driver = MySQL
Server = localhost
Database = asteriskcdrdb
Port = 3306
Socket = /var/run/mysqld/mysqld.sock
Option = 3
EOF
```
## Install FreePBX
```markdown
cd /usr/local/src
wget http://mirror.freepbx.org/modules/packages/freepbx/freepbx-17.0-latest-EDGE.tgz
tar zxvf freepbx-17.0-latest-EDGE.tgz
cd /usr/local/src/freepbx/
./start_asterisk start
./install -n
```
Get the rest of the modules
```markdown
fwconsole ma installall
fwconsole reload
fwconsole restart
```
Set up systemd (startup script)
```markdown
cat <<EOF > /etc/systemd/system/freepbx.service
[Unit]
Description=FreePBX VoIP Server
After=mariadb.service
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/fwconsole start -q
ExecStop=/usr/sbin/fwconsole stop -q
[Install]
WantedBy=multi-user.target
EOF
```
```markdown
systemctl daemon-reload
systemctl enable freepbx
```
## Cradit
1. https://sangomakb.atlassian.net/wiki/spaces/FP/pages/10682545/How+to+Install+FreePBX+17+on+Debian+12+with+Asterisk+21
2. https://wiki.crowncloud.net/?How_to_install_PHP_8_on_Debian_12
3. https://mariadb.com/docs/connectors/mariadb-connector-odbc/building-mariadb-connectorodbc-from-source
