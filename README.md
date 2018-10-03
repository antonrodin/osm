# Crear tu propio servidor Open Street Maps en Ubuntu 16.04

Esto es un manual para crear tu propio servidor para renderizar "tiles" para Open Street Maps (en adelante OSM). Esta orientado a España, ya que 
es un caso un poco especial ya que hay que hacer un "merge" de Islas Canarias y España. Para que el ejemplo sea más rapido de realizar, **se va a realizar con Andorra y en servidores de Arruba** que cuesta aproximadamente 1€ al mes (https://www.arubacloud.com/). Logicamente no voy a publicar la IP de mi servidor real...

Para el servidor real he utilizado un droplet de DigitalOcean para España + Canarias, ocupando un total de 25-30Gb. Dame un eurooooo illo y consigue tus 10$: https://m.do.co/c/2514e298cfc4, ademas al registrarte mete LOWENDBOX y consigues otros 15 dolar frescos. :)

Este tutorial esta basado en esta documentación:

* OSM: https://www.linuxbabe.com/linux-server/openstreetmap-tile-server-ubuntu-16-04
* Apache: https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-16-04
* Osmosis: https://wiki.openstreetmap.org/wiki/Osmosis/Installation
* The problem: https://gis.stackexchange.com/questions/222001/how-to-put-the-two-regions-and-countries-data-on-openstreetmap-into-postgresql
* Leaflet: https://leafletjs.com/
* Lets Encrypt: https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-16-04

## Introducción

Realmente todo viene por el tema de que actualmente los precios de la API Google Street Maps son abusivos (según mi opinión). Existe una alternativa llamada Leaflet.js que utiliza OSM. El problema es que el "Tile Server de OSM" no es gratuito. Pero en unas 2-4 horas puedes montar el tuyo propio que te costara unos 10-15$ al mes y con un rendimiento aceptable. Siempre y cuando solo vas a utilizar el mapa de España. Este manual utilizara **Ubuntu 16.04, Postgre SQL y Servidor web Apache**. Posteriormente intentare crear lo mismo pero con Ubuntu 18.04, aunque puedes hacerlo tu, ya que como puedes comprobar el manual esta en Github con licencia MIT.

## Osmosis:

Instalar Osmosis

```shell
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

Despues de combinar realizamos el paso de importarlo todo en PostgreSQL. *** ¡Ojo! Cuidado con las ruta *** y es posible que te de un *** error de cache ***, es posible que tengas que variar la opcion -C, prueba ponerlo más bajo, ya que si no me equivoco depende de tu RAM.***

```shell
osm2pgsql --slim -d gis -C 3600 --hstore -S ~/openstreetmap-carto-2.41.0/openstreetmap-carto.style merged.pbf
```