# Crear tu propio servidor Open Street Maps en Ubuntu 16.04

Ver 0.1 (En proceso)

Esto es un manual para crear tu propio servidor para renderizar "tiles" para Open Street Maps (en adelante OSM). Esta orientado a España, ya que 
es un caso un poco especial ya que hay que hacer un "merge" de Islas Canarias y España. Para que el ejemplo sea más rapido de realizar, **se va a realizar con Andorra y en servidores de Arruba** que cuesta aproximadamente 1€ al mes (https://www.arubacloud.com/). Logicamente no voy a publicar la IP de mi servidor real...

Para el servidor real he utilizado un droplet de DigitalOcean para España + Canarias, ocupando un total de 25-30Gb. Dame un eurooooo illo y consigue tus 10$: https://m.do.co/c/2514e298cfc4, ademas al registrarte mete LOWENDBOX y consigues otros 15 dolar frescos. :)

Este tutorial esta basado en esta documentación:

* OSM: https://www.linuxbabe.com/linux-server/openstreetmap-tile-server-ubuntu-16-04
* Apache: https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-16-04
* Osmosis: https://wiki.openstreetmap.org/wiki/Osmosis/Installation
* The problem: https://gis.stackexchange.com/questions/222001/how-to-put-the-two-regions-and-countries-data-on-openstreetmap-into-postgresql
* Java JDK: https://www.digitalocean.com/community/tutorials/como-instalar-java-con-apt-get-en-ubuntu-16-04-es
* Leaflet: https://leafletjs.com/
* Lets Encrypt: https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-16-04

## Introducción

Realmente todo viene por el tema de que actualmente los precios de la API Google Street Maps son abusivos (según mi opinión). Existe una alternativa llamada Leaflet.js que utiliza OSM. El problema es que el "Tile Server de OSM" no es gratuito. Pero en unas 2-4 horas puedes montar el tuyo propio que te costara unos 10-15$ al mes y con un rendimiento aceptable. Siempre y cuando solo vas a utilizar el mapa de España. Este manual utilizara **Ubuntu 16.04, Postgre SQL y Servidor web Apache**. Posteriormente intentare crear lo mismo pero con Ubuntu 18.04, aunque puedes hacerlo tu, ya que como puedes comprobar el manual esta en Github con licencia MIT.

## 1. Actualizar.

```shell
sudo apt update
sudo apt upgrade
```

## 2. Instalar PostgreDB con la extensión de PostGis

OSM se cargara en PostgreDB junto con la extensión geoespacial PostGIS. Para instalar:

```shell
sudo apt install postgresql postgresql-contrib postgis postgresql-9.5-postgis-2.2
```

Durante la instalación se creara un usuario **postgres**, cambiamos a este usuario:

```shell
sudo -u postgres -i

#creamos usuario osm
createuser osm

#creamos base de datos 'gis' que pertenece a este usuario con la codificación UTF8
createdb -E UTF8 -O osm gis

#creamos extensiones hstore y gis
psql -c "CREATE EXTENSION hstore;" -d gis
psql -c "CREATE EXTENSION postgis;" -d gis

#salimos
exit
```

## 3. Descarcargar las hojas de estilo y crear usuario osm

Primero creamos usuario osm, que será el responsable de servir, importar las "tiles". Creamos y logueamos:

```shell
sudo adduser osm
su - osm
```

Descargamos las hojas de estilo y descomprimimos

```shell
wget https://github.com/gravitystorm/openstreetmap-carto/archive/v2.41.0.tar.gz
tar xvf v2.41.0.tar.gz
```

Basicamente son las hojas de estilo o estilos por defecto de los mapas. Logicamente se pueden cambiar.

## 5. Osmosis (Caso España):

**¡Ojo!** Si solo vas a importar un pais o el mapa mundi, debes saltar este paso, ya que no es necesario.

Aquí esta el mayor problema de España que no se cubre en otros tutoriales en ingles. Cuando importamos el mapa de España no incluye las Islas Canarias. Islas Canarias por defecto se encuentran en la zona de "Africa". Si no utilizas osmosis el renderizado no funciona correctamente.

Osmosis es una aplicación de línea de comandos hecha en **JAVA** para procesar OSM Data. La necesitaremos para procesar y juntar los datos de cartografía de España e Islas Canarias. Asi que lo primero que tenemos que hacer es **instalar JAVA Java Development Kit, como usuario root**.

```shell
exit
sudo apt-get install default-jdk
```

Volvemos a usuario osm e instalamos osmosis en si:

```shell
su - osm
wget https://bretth.dev.openstreetmap.org/osmosis-build/osmosis-latest.tgz
mkdir osmosis
mv osmosis-latest.tgz osmosis
cd osmosis
tar xvfz osmosis-latest.tgz
rm osmosis-latest.tgz
chmod a+x bin/osmosis
cd bin
./osmosis --help
````

Descargar la cartografía de Andorra y Canarias y combinarla juntos. Logicamente en el caso de España solo tienes que sustituir Andorra por España. La cartografía la puedes descargar gratuitamente de aquí: http://download.geofabrik.de/

```shell
wget -c http://download.geofabrik.de/europe/andorra-latest.osm.pbf
wget -c http://download.geofabrik.de/africa/canary-islands-latest.osm.pbf
```

```shell
./osmosis --read-pbf file="andorra-latest.osm.pbf" --read-pbf file="canary-islands-latest.osm.pbf" --merge --write-pbf file="merged.pbf"
```

La salida tiene esta pinta, es decir en un servidor lamentable combinar andorra y canarias dura unos 24.5 segundos.

```shell
Oct 03, 2018 11:02:24 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Osmosis Version 0.46
Oct 03, 2018 11:02:25 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Preparing pipeline.
Oct 03, 2018 11:02:25 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Launching pipeline execution.
Oct 03, 2018 11:02:25 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline executing, waiting for completion.
Oct 03, 2018 11:02:49 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Pipeline complete.
Oct 03, 2018 11:02:49 AM org.openstreetmap.osmosis.core.Osmosis run
INFO: Total execution time: 24987 milliseconds
```

## 6. Recomendaciones. Configurar el SWAP File y activar el Keep Alive para SSH

Si tenemos poca memoria RAM, como es mi caso y probablemente el tuyo, recomiendo crear un archivo swap. Es muy facil y se hace en cuestión de 20 segundos. **Hay que hacerlo con el usuario root, de ahñi que salimos de osm, con el exit.** Basicamente crea un archivo que ocupa 2 Gb.

```shell
#creamos el archivo en la raiz
sudo fallocate -l 2G /swapfile
#damos permisos solo de usuario root a este archivo
sudo chmod 600 /swapfile
#le damos formato de swap
sudo mkswap /swapfile
#lo activamos
sudo swapon /swapfile
```

Activar el Keep Alive (mantener vivo en spanish), es tambien muy recomendable. Cuando empezamos la importación el proceso dura, diria que al menos 1-2 horas, aunque dependiendo del mapa que vas a importar y la potencia de tu maquina, durarará mas o menos tiempo. Con el Keep Alive activado la conexión SSH con nuestro servidor no se cierra, es decir no te vas a encontrar con el mensaje "Broken Pipe"

```shell
# Abre este archivo con tu editor favorito. Nano, Vim o lo que quieras
sudo vi /etc/ssh/ssh_config
```

Añade esto:

```shell
ServerAliveInterval 60
```

## 7. Importación a PostgreSQL

Este paso recomiendo realizar con al menos un par de horas por delante, ya que dura bastante. Para el mapa de España unas dos horas en un servidor poco potente. Para ello necesitaremos de la herramienta **osm2pgsql** que procesa OSM Map ta y la importa en nuestra base de datos gis creada previamente.

Instalamos (con usuario root):

```shell
sudp apt install osm2pgsql
```

Importamos (con usuario osm):
```shell
su - osm
# Recuerda que nuestro merged file se encuentra en la carpeta de ~/osmosis/bin
# El comando para ejecutar el proceso estando en ese directorio es el siguiente
osm2pgsql --slim -d gis -C 3600 --hstore -S ~/openstreetmap-carto-2.41.0/openstreetmap-carto.style merged.pbf
```

**¡Ojo!** Si te da algun tipo de error relacionado con _cache_ prueba a reducir el parametro "-C 3600" del comando. Suele pasar bastante.

El comando como tal no necesita mucha explicación, es largo pero cada parametro es lógico. Indicamos nuestra base de datos _gis_, junto con la extensión _hstore_, nuestros estilos que hemos descargado y el archivo _merged.pbf_ que hemos generado.

Para importar Andorra + Canarias en una maquina lamentable ha tardado en mi caso 180 segundos. El resultado que te mostrara lo puedes encontrar en el archivo "benchmark-andorra-canarias.rb" en este mismo proyecto

Una vez acabamos la instalación, salimos de usuario osm.

```shell
exit
```

## 8. Instalar mod_tile (usuario root)

El mod_tile, es un modulo del servidor web Apache que se encarga de servir nuestros "tiles". Es necesario compliarlo, pero es muy fácil.

```shell
# Instalar dependencias, entre ellas GIT por ejemplo
sudo apt install git autoconf libtool libmapnik-dev apache2-dev
# Obtener el código fuente
git clone https://github.com/openstreetmap/mod_tile.git
# Compilar e instalar
cd mod_tile
./autogen.sh
./configure
make
sudo make install
sudo make install-mod_tile
```

El proceso tarda unos 5 minutos.

## 9. Generar hoja de estilos para Mapnik

Mapnik si no me equivoco son herramientas de código abierto para generar y basicamente generamos estilos para el.

Primero, instalar dependencias:
```shell
sudo apt install curl unzip gdal-bin mapnik-utils node-carto
```

Generar estilos. (usuario osm)
```shell
su - osm

# Entramos en la carpeta bajada previamente
cd openstreetmap-carto-2.41.0/

# Ejecutamos Script
./get-shapefiles.sh

# Generamos estilos
carto project.mml > style.xml

# Salimos de usuario osm
exit
```

Este procesno no dura mas de 1-2 minutos.

## 10. Configuramos el servicio renderd.

**renderd** es parte de **mod_tile** que hemos instalado antes, una especie de backend que obtiene peticiones del modulo de apache y genera unos archivos .png en nuestro sistema, concretamente enesta carpeta: "_/var/lib/mod_tile_". De hecho se activa como servicio o demonio y en este paso ya es posibible comprobar un poco de todo lo que hemos instalado previamente... que no es poco.

Editamos el archivo de configuración
```shell
sudo vi /usr/local/etc/renderd.conf
```

Bajo la sección __[defaule]__ modificamos esto:

```shell
XML=/home/osm/openstreetmap-carto-2.41.0/style.xml
HOST=localhost
```
Bajo la sección __[mapnik]__ modificmos esto:

```shell
plugins_dir=/usr/lib/mapnik/3.0/input/
```
Guardamos el archivo... e instalamos y configuramos el demonio:

```shell
# Lo copiamos al directorio de inid.t 
sudo cp mod_tile/debian/renderd.init /etc/init.d/renderd
# Le damos permisos de ejecución
sudo chmod a+x /etc/init.d/renderd
```

Configuramos el demonio.

```shell
sudo vi /etc/init.d/renderd

# Cambiamos la siguiente configuración
DAEMON=/usr/local/bin/$NAME
DAEMON_ARGS="-c /usr/local/etc/renderd.conf"
RUNASUSER=osm
```

Como apunte, lo que hemos especificado es que se ejecute como usuario **osm** y le proporcionamos la ruta del archivo de configuración que acabamos de modificar cons los parametros de estilos.

Ahora, creamos una carpeta para nuestros 'tiles' cuyo dueño es el usuario **osm**. Los tiles por cierto son unos archivos .png. De hecho si entradas a esta carpeta pasado un tiempo de funcionamiento de servidor encontraras miles de estos archivos.

```shell
sudo mkdir -p /var/lib/mod_tile
sudo chown osm:osm /var/lib/mod_tile
```

Por último, solo nos queda activar el demonio con estos comandos. Son bastante estandar y puedes hacer el tipico __status, reload, restart.... etc__:

```shell
sudo systemctl daemon-reload
sudo systemctl start renderd
sudo systemctl enable renderd
```

En este paso, podemos comprobar un poco el funcionamiento y realizar cierto debug, puedes ejecutar esto y comprobar:

```shell
root@osm:~# sudo systemctl status renderd
● renderd.service - LSB: Mapnik rendering daemon
   Loaded: loaded (/etc/init.d/renderd; bad; vendor preset: enabled)
   Active: active (running) since Wed 2018-10-03 19:36:03 CEST; 18s ago
     Docs: man:systemd-sysv-generator(8)
   CGroup: /system.slice/renderd.service
           └─6092 /usr/local/bin/renderd -c /usr/local/etc/renderd.conf

Oct 03 19:36:03 osm systemd[1]: Started LSB: Mapnik rendering daemon.
Oct 03 19:36:03 osm renderd[6092]: Loading parameterization function for
Oct 03 19:36:03 osm renderd[6092]: Loading parameterization function for
Oct 03 19:36:03 osm renderd[6092]: Loading parameterization function for
Oct 03 19:36:03 osm renderd[6092]: Loading parameterization function for
Oct 03 19:36:03 osm renderd[6092]: Starting stats thread
Oct 03 19:36:13 osm renderd[6092]: Using web mercator projection settings
Oct 03 19:36:13 osm renderd[6092]: Using web mercator projection settings
Oct 03 19:36:13 osm renderd[6092]: Using web mercator projection settings
Oct 03 19:36:13 osm renderd[6092]: Using web mercator projection settings
```

## 11. Instalar y configurar servidor Web Apache.

```shell
sudo apt install apache2
```

Crear un archivo para cargar el modulo:

```shell
sudo nano /etc/apache2/mods-available/mod_tile.load
```

Pegar la siguiente línea:

```shell
LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so
```

Crear un enlace simbólico:

```shell
sudo ln -s /etc/apache2/mods-available/mod_tile.load /etc/apache2/mods-enabled/
```

Editar el archivo de alojamiento virtual por defecto (default virtual host):

```shell
sudo vi /etc/apache2/sites-enabled/000-default.conf
```

Añadir las siguientes lineas dentro de **<VirtualHost *:80>**

```shell
LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30
```
Reiniciar Servidor Apache

```shell
sudo systemctl restart apache2
```

Para comprobar que todo funcione, teclea en el navegador esto:

```
http://IP_DE_TU_SERVIDOR/osm_tiles/0/0/0.png
```

Nos tendria que aparecer una imagen del mapa mundi en pequeño. Si la ves, es que parece que funciona todo.