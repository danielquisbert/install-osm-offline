# Pasos de instalación OSM offline

## Software y hardware utilizados (requerimientos mínimos)

- S.O.: Linux Debian 9
- Procesador: core i3 (dos núcleos)
- RAM: 4Gb
- HD: 250 GB


## Aplicaciones utilitarias

```
apt-get install unzip curl dh-autoreconf
```

### Instalación PostgreSQL & PostGIS
```
apt-get install postgresql-9.6 postgresql-9.6-postgis-6.1  postgresql-contrib-9.6
```

### configuración de PG
```
su postgres -
createuser -P -s -e osm
```

#### Asumiendo que la dirección IP del servidor es ’192.168.0.1’, añadir a la lista de direcciones junto con ’localhost’ para poder acceder a la base de datos local o remotamente.
```
nano /etc/postgresql/9.6/main/postgresql.conf
```

#### Descomentar la línea 59 y cambiar localhost por *, y guardar
```
listen_addresses = '*'
```

#### Cambiamos los derechos de acceso al servidor PostGreSQL para permitir el acceso del usuario osm desde otra maquina, por ejemplo ’192.168.0.2’, y también desde el propio servidor ’127.0.0.1’.
```
nano -c /etc/postgresql/9.6/main/pg_hba.conf
```
```
#local   all             postgres                                peer
local   all             postgres                                ident
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
#local   all             all                                     peer
local   all             all                                     md5
#IPv4 local connections:

host    all             osm             127.0.0.1/32            md5
host    all             osm             192.168.0.2/32            md5
```

#### Reiniciar Servicios
```
service postgresql restart
service postgresql reload
```

## Instalación Imposm 3
Permite la importación de archivos .osm u osm.pbf dentro de una DB postgresql

### Install dependencias del S.O.
```
apt-get install build-essential git mercurial libgeos-dev
```

### Base de Datos LevelDB
Es una DB open source NoSQL, no soporta el modelo relacional, ni sql e índices
```
mkdir /opt/library && cd /opt/library
git clone https://github.com/danielquisbert/leveldb.git
cd leveldb
make
cp -r include/leveldb /usr/include/
cp out-shared/libleveldb.* /usr/lib/
ldconfig
```
### Lenguaje GOv1.8.3
```
cd /opt
wget -c https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
tar -xvf go1.8.3.linux-amd64.tar.gz
cp -r go /usr/local/
nano /root/.profile
export PATH=$PATH:/usr/local/go/bin
```
#### **OjO: Además lanzarlo desde la terminal: export PATH=$PATH:/usr/local/go/bin**

### Install Levigo
Es es un contenedor para LevelDB. 
Configurar variables de entorno “CGO_CFLAGS, CGO_LDFLAGS” que apunten al directorio donde se encuentra instalado levelDB.
Seteamos el contenedor
CGO_CFLAGS="-I/opt/library/leveldb/include" CGO_LDFLAGS="-L/opt/library/leveldb" 

```
go get github.com/jmhodges/levigo
```

#### Install GO-SQLITE3
```
go get github.com/mattn/go-sqlite3
```

### Install Protocol Buffers
```
cd /opt/library
git clone https://github.com/danielquisbert/protobuf.git
cd protobuf
./autogen.sh
./configure
make
make install
```
```
go get -u github.com/golang/protobuf/protoc-gen-go
```

#### Install PQ
Es el controlador postgres para el lenguaje Go
```
go get github.com/lib/pq
```

### Compilación e instalación
```
mkdir /opt/imposm && cd /opt/imposm
export GOPATH=$PWD
go get github.com/omniscale/imposm3

go install github.com/omniscale/imposm3/cmd/imposm3
```
```
echo $GOPATH
```
El resultado debe ser: /opt/imposm
```
cp -r /opt/imposm/* /usr/local/go/

imposm3 version
```

## Creación del usuario y la DB
```
su postgres -
createuser -P -s -e imposmuser
createdb imposm -O imposmuser

nano -c /etc/postgresql/9.6/main/pg_hba.conf
service postgresql restart
service postgresql reload

su postgres -
createlang -h 127.0.0.1 -U imposmuser plpgsql imposm
psql -h 127.0.0.1 -U imposmuser -d imposm -f /usr/share/postgresql/9.6/contrib/postgis-3.1/postgis.sql
psql -h 127.0.0.1 -U imposmuser -d imposm -f /usr/share/postgresql/9.6/contrib/postgis-3.1/spatial_ref_sys.sql
```

## Importación del archivo de datos OSM
```
mkdir -p /opt/OSM/imposm && cd /opt/OSM
mkdir /opt/OSM/cache

imposm3 import -config config-imposm.json -appendcache -deployproduction -read ../south-america-latest.osm.pbf  -write
```
## Generación del fondo de mapa
### Mapserver
### instalación de paquetes necesarios
```
apt-get install libpng12-0 libfreetype6 libgd2-xpm-dev zlib-gst libproj0 libcurl3 libagg-dev libgdal1-dev libgeotiff-dev libjpeg-dev libgeos-dev libxml2-dev libpq-dev libgd2-xpm-dev libpng12-dev libfreetype6-dev zlib1g-dev libproj-dev gdal-bin libgdal-dev libgdal1-dev libfcgi-dev cmake libfribidi-bin libharfbuzz-dev libcairo-dev swig

mkdir -p /opt/mapserver && cd /opt/mapserver
wget -c http://download.osgeo.org/mapserver/mapserver-7.0.5.tar.gz
tar -xvf mapserver-7.0.5.tar.gz
cd mapserver-7.0.5
mkdir build
cd build
```
```
cmake -DCMAKE_INSTALL_PREFIX=/opt \
        -DCMAKE_PREFIX_PATH=/usr/local:/opt  \
        -DWITH_CLIENT_WFS=ON \
        -DWITH_CLIENT_WMS=ON \
        -DWITH_CURL=ON \
        -DWITH_SOS=ON \
        -DWITH_FRIBIDI=0 \
        -DWITH_HARFBUZZ=0 \
        ../ >../configure.out.txt

make
make install
```


## Install estilos
```
cd /opt/OSM
git clone https://github.com/danielquisbert/basemaps.git
git branch --track 6.2 origin/branch-6-2
git checkout 6.2
```


## Install Apache2
```
apt-get install -y apache2 apache2-mpm-worker libapache2-mod-fastcgi
a2enmod actions fastcgi alias
apt-get install libapache2-mod-php7.0 php7-common php7-cli php7-fpm php7.0
```
```
cp /opt/bin/mapserv /usr/lib/cgi-bin/

mkdir /tmp/ms_tmp
chmod 777 /tmp/ms_tmp

nano /etc/rc.local
mkdir /tmp/ms_tmp
chmod 777 /tmp/ms_tmp
```


### Probar en el navegador
```
http://localhost/cgi-bin/mapserv?map=/opt/OSM/basemaps/osm-google.map&mode=browse&template=openlayers&layers=all
```
