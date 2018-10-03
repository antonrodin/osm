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

## 5. Osmosis:

Osmosis es una aplicación de línea de comandos hecha en **JAVA** para procesar OSM Data. La necesitaremos para juntar los datos de cartografía de España e Islas Canarias. Asi que lo primero que tenemos que hacer es **instalar JAVA Java Development Kit, como usuario root**.

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

Descargar la cartografía de Andorra y Canarias y combinarla juntos. Logicamente en el caso de España solo tienes que sustituir...

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

Despues de combinar realizamos el paso de importarlo todo en PostgreSQL. *** ¡Ojo! Cuidado con las ruta *** y es posible que te de un *** error de cache ***, es posible que tengas que variar la opcion -C, prueba ponerlo más bajo, ya que si no me equivoco depende de tu RAM.***

```shell
osm2pgsql --slim -d gis -C 3600 --hstore -S ~/openstreetmap-carto-2.41.0/openstreetmap-carto.style merged.pbf
```