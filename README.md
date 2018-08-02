# oregon-choropleth

## Resources

County-level data for Oregon:
- https://www.census.gov/cgi-bin/geo/shapefiles/index.php?year=2010&layergroup=Counties+%28and+equivalent%29

Preview shapefiles:
- http://mapshaper.org/

Geographic projections:
- https://github.com/veltman/d3-stateplane
- http://spatialreference.org/

## Procedure

Fetch Oregon county shapes
```
mkdir formatted raw
mv ~/Downloads/tl_2010_41_county10.zip ./raw/
unzip -o raw/tl_2010_41_county10.zip
```

Fetch population data by county
```
curl 'https://api.census.gov/data/2015/acs5?get=B01003_001E&for=county:*&in=state:41' -o raw/or-county-populations.json
```

Convert county data JSON array to an NDJSON stream
- `ndjson-cat`: remove newlines
- `ndjson-split`: split into multiple lines
- `ndjson-map`: reformat each line as an object

```
ndjson-cat raw/or-county-populations.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[2], population: d[0]}' \
  > formatted/or-county-populations.ndjson
```

Convert shapefile to GeoJSON and apply a geographic projection
```
shp2json raw/tl_2010_41_county10.shp \
    | geoproject 'd3.geoConicConformal().parallels([44 + 20 / 60, 46]).rotate([120 + 30 / 60, 0]).fitSize([960, 960], d)' \
    > formatted/or-projected.json
```

Preview in a browser
```
geo2svg -w 960 -h 960 < formatted/or-projected.json > formatted/or-projected.svg
```

Convert the GeoJSON feature collection to a newline-delimited stream of GeoJSON features,
then set each feature’s id using `ndjson-map`:
```
ndjson-split 'd.features' \
  < formatted/or-projected.json \
  | ndjson-map 'd.id = d.properties.COUNTYFP10, d' \
  > formatted/or-projected-id.ndjson

```

Join the population data to the geometry using ndjson-join,
and remove the props we don't need
```
ndjson-join 'd.id' formatted/or-county-populations.ndjson formatted/or-projected-id.ndjson \
  | ndjson-map 'd[1].properties = {population: d[0].population}, d[1]' \
  > formatted/or-joined.ndjson
```

Now, perform some topology-preserving simplification, topoquantize, and delta-encode
```
geo2topo -n \
  counties=formatted/or-joined.ndjson \
  | topoquantize 1e5 \
  > formatted/or-topo.json
```

We need a non-linear transform to distribute the colors equitably, so we'll use a threshold scale.
Use `topo2geo` to extract the simplified counties from the topology,
pipe to `ndjson-map` to assign the fill property for each county,
pipe to `ndjson-split` to break the collection into features,
and lastly pipe to `geo2svg`
```
topo2geo counties=- \
  < formatted/or-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1000, 10000, 25000, 50000, 100000, 300000, 600000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.population)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > formatted/or-counties-threshold.svg
```
