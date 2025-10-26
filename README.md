# OSM Tile Server

An example project that demonstrates how to quickly setup an Openstreetmap Tile Server for a small region with docker compose.

This specific setup assumes that a region is used instead of the whole world and that a `.poly` file for this region is provided. This allows auto-updates of the data as well as pre-rendering of tiles for this region.

## Setup

Create the necessary folders:

```shell
mkdir data tiles pbf
```

### Prepare environment settings and data

Create the `.env` file (make sure to adjust the `PBF_URL`):

```shell
# Replace with the desired URLs
PBF_URL=https://download.geofabrik.de/europe/germany/berlin-latest.osm.pbf
POLY_URL=https://download.geofabrik.de/europe/germany/berlin.poly

tee .env > /dev/null << EOF
PBF_URL=${PBF_URL}
POLY_URL=${POLY_URL}
PBF_FILE=$(basename "${PBF_URL}")
POLY_FILE=$(basename "${POLY_URL}")
EOF
```

Download the region of interest:

```shell
source .env
wget -q --show-progress -O pbf/${PBF_FILE} ${PBF_URL}
wget -q --show-progress -O pbf/${POLY_FILE} ${POLY_URL}
```

### Import data

May take a long time (up to multiple days, depending on the size of the `.pbf` file and therefore the region it covers, as well as on the performance and resources of the server)!

Start the import and wait until it is finished and exits automatically:

```shell
docker compose up tiles-import
```

### (Optional) Pre-render tiles for the downloaded region

Setup scripts and env vars:

```shell
POLY_FILE=data/region.poly

RENDER_GEO_URL=https://raw.githubusercontent.com/alx77/render_list_geo.pl/master/render_list_geo.pl
RENDER_GEO_FILE=data/$(basename "${RENDER_GEO_URL}")
wget -q -O ${RENDER_GEO_FILE} ${RENDER_GEO_URL}
chmod +x ${RENDER_GEO_FILE}

POLY2BBOX_URL=https://raw.githubusercontent.com/openstreetmap/svn-archive/main/applications/utils/osm-extract/polygons/poly2bb.pl
POLY2BBOX_FILE=data/$(basename "${POLY2BBOX_URL}")
wget -q -O ${POLY2BBOX_FILE} ${POLY2BBOX_URL}
chmod +x ${POLY2BBOX_FILE}
```

Test whether extracting the BBOX works (open the returned url):

```shell
# export returned string "left=... right=... top=... bottom=..." to env vars left, right, top, bottom
for kv in $($POLY2BBOX_FILE $POLY_FILE); do
  export "${kv%%=*}"="${kv#*=}"
done

echo "https://tools.geofabrik.de/calc/#type=geofabrik_standard&bbox=$left,$bottom,$right,$top&tab=1&proj=EPSG:4326&places=2"
```

Render all tiles for the given bbox and zoom range:

```shell
MIN_ZOOM=0
MAX_ZOOM=12
N_THREADS=20

RENDER_GEO_FILE_DOCKER=/data/database/$(basename "${RENDER_GEO_URL}")
BBOX=$($POLY2BBOX_FILE $POLY_FILE | sed -E 's/left=/-x /; s/right=/-X /; s/top=/-Y /; s/bottom=/-y /')
docker compose exec -it tiles bash -c "$RENDER_GEO_FILE_DOCKER $BBOX -a -z $MIN_ZOOM -Z $MAX_ZOOM -n $N_THREADS"
```

Render all tiles for the whole world for lower resolution zoom levels:

```shell
MIN_ZOOM=0
MAX_ZOOM=8
N_THREADS=20

docker compose exec -it tiles bash -c "render_list -a -z $MIN_ZOOM -Z $MAX_ZOOM -n $N_THREADS"
```

## Usage

Start the tileserver at http://127.0.0.1:8080/ with an exemplary implementation with **leaflet**:

```shell
docker compose up -d tiles
```

## Future Ideas

- [improve performance of pre-rendering tiles](https://gis.stackexchange.com/questions/415340/pre-rendering-tile-using-render-list-command-is-extremely-slow)

## Resources

- [CoWayger on Github - how to prerender tiles in 2022 (for a specific area)](https://github.com/Overv/openstreetmap-tile-server/issues/343#issuecomment-1313522862)
- [Openstreetmap Tile Server](https://github.com/Overv/openstreetmap-tile-server)
- [Geofabrik - Baden-Wuerttemberg](https://download.geofabrik.de/europe/germany/baden-wuerttemberg.html)
- [Bytepark - Einen eigenen OpenStreetMap Tileserver betreiben](https://www.bytepark.de/blog/einen-eigenen-openstreetmap-tileserver-betreiben)
- [switch2osm.org - serving your own tiles](https://switch2osm.org/serving-tiles/)