Este es el resultado de la importación de Andorra + Canarias con la herramienta **osm2pgsql**

```
osm@osm:~/osmosis/bin$ osm2pgsql --slim -d gis -C 3600 --hstore -S ~/openstreetmap-carto-2.41.0/openstreetmap-carto.style merged.pbf
osm2pgsql SVN version 0.88.1 (64bit id space)

Using built-in tag processing pipeline
Using projection SRS 900913 (Spherical Mercator)
Setting up table: planet_osm_point
Setting up table: planet_osm_line
Setting up table: planet_osm_polygon
Setting up table: planet_osm_roads
Allocating memory for dense node cache
Allocating dense node cache in one big chunk
Allocating memory for sparse node cache
Sharing dense sparse
Node-cache: cache=3600MB, maxblocks=460800*8192, allocation method=11
Mid: pgsql, scale=100 cache=3600
Setting up table: planet_osm_nodes
Setting up table: planet_osm_ways
Setting up table: planet_osm_rels

Reading in file: merged.pbf
Processing: Node(4599k 230.0k/s) Way(488k 16.83k/s) Relation(10350 398.08/s)  parse time: 75s
Node stats: total(4599587), max(5949396617) in 20s
Way stats: total(488114), max(629954927) in 29s
Relation stats: total(10351), max(8752242) in 26s
Committing transaction for planet_osm_point
Committing transaction for planet_osm_line
Committing transaction for planet_osm_polygon
Committing transaction for planet_osm_roads
Setting up table: planet_osm_nodes
Setting up table: planet_osm_ways
Setting up table: planet_osm_rels
Using built-in tag processing pipeline

Going over pending ways...
	257576 ways are pending

Using 1 helper-processes
Finished processing 257576 ways in 66 sec

257576 Pending ways took 66s at a rate of 3902.67/s
Committing transaction for planet_osm_point
Committing transaction for planet_osm_line
Committing transaction for planet_osm_polygon
Committing transaction for planet_osm_roads

Going over pending relations...
	0 relations are pending

Using 1 helper-processes
Finished processing 0 relations in 0 sec

Committing transaction for planet_osm_point
WARNING:  there is no transaction in progress
Committing transaction for planet_osm_line
WARNING:  there is no transaction in progress
Committing transaction for planet_osm_polygon
WARNING:  there is no transaction in progress
Committing transaction for planet_osm_roads
WARNING:  there is no transaction in progress
Stopping table: planet_osm_rels
Building index on table: planet_osm_rels
Sorting data and creating indexes for planet_osm_line
Stopping table: planet_osm_ways
Building index on table: planet_osm_ways
Stopping table: planet_osm_nodes
Sorting data and creating indexes for planet_osm_point
Stopped table: planet_osm_nodes in 0s
Sorting data and creating indexes for planet_osm_roads
Sorting data and creating indexes for planet_osm_polygon
Stopped table: planet_osm_rels in 2s
Copying planet_osm_roads to cluster by geometry finished
Creating geometry index on  planet_osm_roads
Creating osm_id index on  planet_osm_roads
Creating indexes on  planet_osm_roads finished
All indexes on  planet_osm_roads created  in 3s
Completed planet_osm_roads
Copying planet_osm_point to cluster by geometry finished
Creating geometry index on  planet_osm_point
Creating osm_id index on  planet_osm_point
Copying planet_osm_polygon to cluster by geometry finished
Creating geometry index on  planet_osm_polygon
Creating indexes on  planet_osm_point finished
All indexes on  planet_osm_point created  in 18s
Completed planet_osm_point
Copying planet_osm_line to cluster by geometry finished
Creating geometry index on  planet_osm_line
Creating osm_id index on  planet_osm_polygon
Creating indexes on  planet_osm_polygon finished
All indexes on  planet_osm_polygon created  in 29s
Completed planet_osm_polygon
Creating osm_id index on  planet_osm_line
Creating indexes on  planet_osm_line finished
All indexes on  planet_osm_line created  in 33s
Completed planet_osm_line
Stopped table: planet_osm_ways in 39s
node cache: stored: 4599587(100.00%), storage efficiency: 52.19% (dense blocks: 1235, sparse nodes: 3774658), hit rate: 100.00%

Osm2pgsql took 180s overall
osm@osm:~/osmosis/bin$ 

```