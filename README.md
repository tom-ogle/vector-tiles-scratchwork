`osm-pbf-data/south-yorkshire-latest.osm.pbf` is data from OpenStreetMap and https://www.geofabrik.de/ 


Vector tiles are used by many in-browser/app renderers, and can also power server-side raster rendering. They enable on-the-fly style changes and greater interactivity, while imposing less of a storage burden. You can output them to individual files, or to a SQLite (.mbtiles) database.



# Contents

* Cut down area in a pbf
  * osmium
* Load OSM data into PostGIS
  * imposm3
  * osm2pgsql
  * ogr2ogr
* Map updates into PostGIS
* Generate Vector Tiles
  * Directly from OSM PBF
  * Via PostGIS


## Cut down area in a pbf

### osmium

```bash
osmium extract --bbox=2.68,50.72,7.55,54.12 --set-bounds --strategy=....osm.pbf  --output ....osm.pbf
```


## Load OSM data into PostGIS

### imposm3

Uses web mercator projection (EPSG:3857) by default.

* Ensure imposm3 is installed
* `imposm import -connection postgis://postgres:password@localhost/db \
  -mapping /path/to/mapping.json -read /path/to/osm.pbf -write`
* Once the import is finished a file called `last.state.txt` in `/home/osm/data/imposm3_cache`. This can be used by 
 osmosis for updates. 

Import data into the "import" schema:
`imposm3 import -config config.json -read planet-latest.osm.pbf -write -optimize -overwritecache -diff`
Move to production schema:
`imposm3 import -config config.json -deployproduction`

#### Via Docker

##### Use imposm3 default mapping
```bash
docker run --rm \
    --rm \
    --network=vector-tiles-scratchpad_vector-tiles \
    -v $(pwd)/osm-pbf-data:/import \
    -v $(pwd)/import-osm/mapping.yml:/mapping/mapping.yaml \
    -e PGHOST="postgis" \
    -e PGDATABASE="imposm3" \
    -e PGUSER="postgres" \
    -e PGPASSWORD="password" \
    openmaptiles/import-osm:4.1.0
```

##### Use openmaptiles generated mapping
```bash
docker run --rm \
    --rm \
    --network=vector-tiles-scratchpad_vector-tiles \
    -v $(pwd)/osm-pbf-data:/import \
    -v $(pwd)/import-osm/openmaptiles-mapping.yaml:/mapping/mapping.yaml \
    -e PGHOST="postgis" \
    -e PGDATABASE="openmaptiles" \
    -e PGUSER="postgres" \
    -e PGPASSWORD="password" \
    openmaptiles/import-osm:4.1.0
```

#### osm2pgsql

Uses web mercator projection (EPSG:3857) by default.

###### output and style

Use --output / -O for output and --style / -S for style.

Output is one of:
* pgsql - default output back-end and is optimized for rendering with Mapnik
* flex - allows more flexible configuration
* gazetteer - intended for geocoding with Nominatim
* null - does not write any output and is only useful for testing or with --slim for creating slim tables.

For the pgsql output, the default style is /usr/share/osm2pgsql/default.style, for other outputs there is no default.

osm2pgsql --style /openstreetmap-carto/openstreetmap-carto.style -d gis -U postgres -k --slim /great-britain-latest.osm.pbf
osm2pgsql -G --hstore --style /openstreetmap-carto/openstreetmap-carto.style --slim --host $POSTGIS_HOST -d gis -U postgres /Melbourne.osm.pbf
osm2pgsql --slim -d gis -C 3600 --tag-transform-script /openstreetmap-carto/openstreetmap-carto.lua --hstore -S /openstreetmap-carto/openstreetmap-carto.style --host $POSTGIS_HOST -U postgres /Melbourne.osm.pbf
osm2pgsql --slim -d gis -C 3600 --hstore -S /openstreetmap-carto/openstreetmap-carto.style --host $POSTGIS_HOST -U postgres --tag-transform-script /openstreetmap-carto/openstreetmap-carto.lua /Melbourne.osm.pbf

Params:

* --slim: Store temporary data in the database. Without this mode, all temporary data is stored in RAM and if you do not have enough the import will not work successfully
* --hstore: Add tags without column to an additional hstore (key/value) column in the database tables.
* --output: Specifies the output back-end to use: pgsql, flex, gazetteer or null.
* --tag-transform-script: Specify a Lua script to handle tag filtering and normalisation. The script contains callback functions for nodes, ways and relations, which each take a set of tags and returns a transformed, filtered set of tags which are then written to the database.

###### Import using osm2pgsql default
```bash
PGPASSWORD=password osm2pgsql \
    --host localhost \
    --port 5432 \
    --database osm2pgsql_default \
    --username postgres \
    --slim \
    --hstore \
    --output pgsql \
    ./osm-pbf-data/south-yorkshire-latest.osm.pbf
```

###### Import using openstreetmap-carto style
```bash
PGPASSWORD=password osm2pgsql \
    --host localhost \
    --port 5432 \
    --database osm2pgsql_openstreetmap_carto \
    --username postgres \
    --slim \
    --hstore \
    --output pgsql \
    --tag-transform-script "./osm2pgsql/openstreetmap-carto/openstreetmap-carto.lua" --style "./osm2pgsql/openstreetmap-carto/openstreetmap-carto.style" \
    ./osm-pbf-data/south-yorkshire-latest.osm.pbf
```

#### ogr2ogr

ogr2ogr has multiple drivers for handling different data:

* mvt - https://gdal.org/drivers/vector/mvt.html#vector-mvt
* mbtiles - https://gdal.org/drivers/raster/mbtiles.html
* osm pbf/xml - https://gdal.org/drivers/vector/osm.html (can't create, only input)
* PostgreSQL / PostGIS - https://gdal.org/drivers/vector/pg.html
* geojson - https://gdal.org/drivers/vector/geojson.html
* geotiff - https://gdal.org/drivers/raster/gtiff.html
* gpx - https://gdal.org/drivers/vector/gpx.html

Not tested yet:
```bash
ogr2ogr -f PostgreSQL PG:"host=localhost user=postgres dbname=ogr2ogr password=password port=5432" \
    ./osm-pbf-data/south-yorkshire-latest.osm.pbf \
    -lco COLUMN_TYPES=other_tags=hstore \
    --config OSM_MAX_TMPFILE_SIZE 1024
```


### Map updates into PostGIS

## osmosis: 


## Generate Vector Tiles

### Directly from OSM PBF

#### Tilemaker

Tilemaker generates vector tiles (in Mapbox Vector Tile format) from OSM PBF format files.

* https://wiki.openstreetmap.org/wiki/Tilemaker
* https://tilemaker.org/
* https://github.com/systemed/tilemaker
* Permissive license!


* Download https://osmdata.openstreetmap.de/download/water-polygons-split-4326.zip and unzip so `coastline/water_polygons.shp` can be found.
* Build tilemaker from the github repo `https://github.com/systemed/tilemaker` so `tilemaker/bin/tilemaker` can be found.

--config: A JSON file listing each layer, and the zoom levels at which to apply it
--process: A Lua program that looks at each node/way's tags, and places it into layers accordingly

```bash
cd tilemaker &&
./bin/tilemaker --input ../osm-pbf-data/south-yorkshire-latest.osm.pbf \
    --config ./resources/config-openmaptiles.json \
    --process ./resources/process-openmaptiles.lua \
    --output ../serve/tiles
```


### Via PostGIS

