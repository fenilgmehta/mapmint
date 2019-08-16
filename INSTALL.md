# ![logo](mapmint-ui/img/mapmint-logo-small.png "MapMint") Installation guide

All the instructions given in the following suppose that you are logged in
as a basic UNIX user, not root.

<h3>Install all the dependencies (Ubuntu 14.04 LTS)</h3>

```sh
# NOTE: if you encounter an error saying "Unable to locate package" or "Package '...' has no installation candidate", then remove them from the list and repeat the step.
sudo apt-get update
INSTALL_PACKAGE_LIST='flex bison libfcgi-dev libxml2 libxml2-dev curl
openssl autoconf apache2 software-properties-common subversion git
libmozjs185-dev python3-dev build-essential libfreetype6-dev
libproj-dev libgdal1-dev libcairo2-dev apache2-dev libxslt1-dev
python3-cheetah cssmin python3-psycopg2 python-gdal python-libxslt1
postgresql-9.3 r-base cmake gdal-bin libapache2-mod-fcgid ghostscript
xvfb net-tools libreoffice libreoffice-common'
for pkg in $INSTALL_PACKAGE_LIST
do
	# echo $pkg
    sudo apt-get install -m $pkg
done

pip3 install Cheetah3
```

<h3>Initial settings</h3>

We will refer to the ```$SRC``` variable in all step of this short documentation, so make sure to run the following command and use the same terminal until the end of the setup process.

```sh
export SRC=/home/djay/src
mkdir -p $SRC
```

<h3>Download ZOO-Project and MapMint</h3>

```sh
cd $SRC
svn checkout http://www.zoo-project.org/svn/trunk zoo
git clone https://github.com/mapmint/mapmint.git
```


<h3>Install MapServer</h3>

```sh
cd $SRC
wget http://download.osgeo.org/mapserver/mapserver-6.2.0.tar.gz
tar -xvf mapserver-6.2.0.tar.gz

cd mapserver-6.2.0
./configure --with-wfs --with-python --with-freetype=/usr/ --with-ogr --with-gdal --with-proj --with-geos --with-cairo --with-kml --with-wmsclient --with-wfsclient --with-wcs --with-sos --with-python=/usr/bin/python3.6 --without-gif --with-apache-module --with-apxs=/usr/bin/apxs2 --with-apr-config=/usr/bin/apr-1-config --enable-python-mapscript --with-zlib --prefix=/usr/
sed "s:mapserver-6.2.0-mm/::g;s:mapserver-6.2.0/::g" -i ../mapmint/thirds/ms-6.2.0-full.patch
patch -p0 < $SRC/mapmint/thirds/ms-6.2.0-full.patch 
patch mapscript/python/Makefile <<< '26,27c
PYLIBDIR=`$(PYTHON) -c "from distutils.sysconfig import get_python_lib; print(get_python_lib(1))"`
PYINCDIR=`$(PYTHON) -c "from distutils.sysconfig import get_python_inc; print(get_python_inc(1))"`
.'
patch mapscript/python/Makefile.in <<< '26,27c
PYLIBDIR=`$(PYTHON) -c "from distutils.sysconfig import get_python_lib; print(get_python_lib(1))"`
PYINCDIR=`$(PYTHON) -c "from distutils.sysconfig import get_python_inc; print(get_python_inc(1))"`
.'

make
sudo make install
```

<h3>Install MapCache</h3>

```sh
cd $SRC
git clone https://github.com/mapserver/mapcache.git
cd mapcache/
cmake .
make 
sudo make install

wget http://geolabs.fr/dl/mapcache.xml
sudo cp mapcache.xml /usr/lib/cgi-bin/
```

Create or edit the ```/etc/ld.so.conf.d/zoo.conf``` and add the line ```/usr/local/lib``` manually or run the following line:
```sh
sudo echo "/usr/local/lib" > /etc/ld.so.conf.d/zoo.conf
```

then run the following command:

```sh
sudo ldconfig
```

<h3>Install ZOO-Project and the C Services</h3>

```sh
cd $SRC/zoo/thirds/cgic206
sed "s:lib64:lib:g" -i Makefile 
make
cd $SRC/zoo/zoo-project/zoo-kernel
autoconf
./configure ./configure --with-mapserver=$SRC/mapserver-6.2.0/ --with-python --with-pyvers=3.6 --with-js=/usr/ --with-xsltconfig=/usr/bin/xslt-config
sed "s:/usr/lib/x86_64-linux-gnu/libapr-1.la::g" -i ZOOMakefile.opts
make
make install
ldconfig
cp zoo_loader.cgi ../../../mapmint/mapmint-services/

cd $SRC/zoo/zoo-project/zoo-services/ogr/ogr2ogr
make
cp cgi-env/* $SRC/mapmint/mapmint-services/vector-converter/

cd $SRC/zoo/zoo-project/zoo-services/ogr/base-vect-ops
make
cp cgi-env/* $SRC/mapmint/mapmint-services/vector-tools/

cd $SRC/zoo/zoo-project/zoo-services/gdal
for i in contour dem grid profile translate warp ; do
echo $i
cd $i
make; sudo cp cgi-env/* $SRC/mapmint/mapmint-services/raster-tools/
cd ..
done
```

<h3>Build MapMint C Services</h3>

```sh
cd $SRC/mapmint/mapmint-services
for i in *-src ; do
echo $i
cd $i
autoconf
./configure --with-zoo-kernel=$SRC/zoo/zoo-project/zoo-kernel --with-mapserver=$SRC/mapserver-6.2.0
make
cd ..
done
```

<h3>Start LibreOffice as server</h3>

First start a screen named ```PaperMint```:

```sh
screen -r -R PaperMint
```

From this screen, run the following commands:

```sh
Xvfb :11&
export DISPLAY=:11
soffice --nofirststartwizard --norestore --nocrashreport --headless "--accept=socket,host=127.0.0.1,port=3662;urp"
```

Now you can check in another terminal if the server is available,
press ```CTRL+a``` then ```c``` from  your screen to open a new
terminal. Then run the following command:

```sh
netstat -na | grep 3662
```

You should see the 3662 as an open port.

Then press ```CTRL+a``` then ```d``` to quit the screen but keeping it
available for future use.

<h3>Install QREncode service</h3>

```sh
cd $SRC
wget http://fukuchi.org/works/qrencode/qrencode-3.4.1.tar.gz
tar xvf qrencode-3.4.1.tar.gz
cd qrencode-3.4.1
./configure
make 
sudo make install
sudo ldconfig -v
cd $SRC/zoo/zoo-project/zoo-services/qrencode/
make
sudo cp cgi-env/* $SRC/mapmint/mapmint-services/
```

<h3>Install R</h3>

MapMint use specific R packages for giving the administrator access to discretisation options in the Styler window.

```sh
sudo R
install.packages("e1071")
install.packages("classInt")
q()
```

<h3>Final tweeks</h3>

```sh
sudo ln -s $SRC/mapmint/mapmint-ui/ /var/www/html/ui
sudo ln -s $SRC/mapmint/public_map/ /var/www/html/pm

sudo ln -s $SRC/mapmint/mapmint-services/ /usr/lib/cgi-bin/mm

sudo a2enmod fcgid
sudo a2enmod cgid
sudo a2enmod rewrite
```

Edit the apache2 configuration file with the following command:
```sh
sudo vi /etc/apache2/apache2.conf 
```

Then make sure the Directory block for /var/www looks like the following, in other case correct it:

```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>

FcgidInitialEnv "MAPCACHE_CONFIG_FILE" "/usr/lib/cgi-bin/mapcache.xml"
ScriptAlias /cache      /usr/lib/cgi-bin/mapcache.fcgi

<Location /cache>
        Order Allow,Deny
        Allow from all
        SetHandler fcgid-script
</Location>

```

Edit the file ```/etc/apache2/conf-available/serve-cgi-bin.conf```,
then replace ```+SymLinksIfOwnerMatch``` by ```+FollowSymLinks```.

Restart the apache web server

```sh
sudo /etc/init.d/apache2 restart
```

Download and use databases required by MapMint, you are invited to
load the sql files related to the PostGIS module before loading the
mmdb.sql file (the location will depend on your setup).
```sh
sudo mkdir /var/data
cd /var/data
sudo wget http://geolabs.fr/dl/mm.db

sudo wget http://geolabs.fr/dl/mmdb.sql

sudo /etc/init.d/postgresql start

createdb -E utf-8 mmdb
psql mmdb -f mmdb.sql
```

Create required directories

```sh
sudo cp -r $SRC/mapmint/template/data/* /var/data
sudo mkdir /var/data/{templates,dirs,public_maps,georeferencer_maps}
sudo mkdir -p /var/www/html/tmp/descriptions
sudo mkdir -p /var/www/html/pm/styles

cd $SRC
wget http://geolabs.fr/dl/fonts.tar.bz2
sudo mkdir /var/data/fonts
sudo tar -xvf fonts.tar.bz2 -C /var/data/fonts

cp $SRC/mapmint/mapmint-ui/js/.htaccess $SRC/mapmint/public_map/
sudo chown -R www-data:www-data /var/data
sudo chown www-data:www-data /usr/lib/cgi-bin/mm/main.cfg
sudo chown www-data:www-data /usr/lib/cgi-bin/mapcache.xml
sudo chown -R www-data:www-data /var/www/html/pm/styles

```

Now you should edit the ```main.cfg``` file located in
```$SRC/mapmint/mapmint-services``` to fit with your setup, you can
have more informations about its content from here: https://github.com/mapmint/mapmint/blob/master/Main_cfg.md. Then you should be ready to access your new MapMint installation through the following url: http://[HOST]:[PORT]/ui/Dashboard .

Initially, your admin login is ```test``` and the password is ```demo02```. You are invited to remove this defalut account from the admin user interface.

You should add at least one datastore in the ```/var/data/ftp/``` directory by creating new directory inside it and add your GIS data in this sub-directory.
