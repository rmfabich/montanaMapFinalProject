This map could not have been created without Mike Bostock's in depth shared knowledge on map builing.
Created a choropleth map of Montana's population density from the command line using d3-geo, TopoJSON and ndjson-cli (newline delimited JSON). Already had node & npm on my mac using Homebrew. Also installed curl and unzip.
The population estimates are per census tracts, so I need census tract polygons. I used the following URL: https://www2.census.gov/geo/tiger/GENZ2017/shp/
Had to look up Montana's FIPS code (30 for Montana) then used curl to download: curl 'http://ww2.census.gov/gov/geo/tiger/GENZ2017/shp/cb_2017_30_tract_500k.zip' -0 cb_2017_30_tract_500k.zip
Next upzip the file: unzip -o cb_2017_06_tract-500k.zip
Binary shapefiles are difficult to work with so convert to GeoJSON. Install: npm install -g shapefile. Now use shp2json to convert to GeoJSON: shp2json cb_2017_mt_tract_500k.shp -o mt.json
Then I applied a geographic projection by installling: npm install -g d3-geo-projection
Use this site to find what projection to use for each state https://github.com/veltman/d3-stateplane. I used geoproject: geoproject 'd3.geoConicconformal().parallels([45, 49]).rotate([109 + 30 / 60, 0]).fitSize([960, 960], d)' < mt.json > mt-albers.json
To preview the projected geometry use d3-geo-projection's geo2svg: geo2svg -w 960 -h 960 < mt-albers.json > mt-albers.svg
Installed newline-delimited JSON so that each line is a valid JSON value. Install: geo2svg -w 960 -h 960 < mt-albers.json > mt-albers.svg
Then convert the collection to a newline-delimited stream of GeoJSON features, use: ndjson-split 'd.features' \
  < mt-albers.json \
  > mt-albers.ndjson
I had a little trouble with this in the command line and had to remove the back slashes.
We now have one feature (one census tract) per line. We can now manipulate individual features. 
Now set each feature's id using ndjson-map: ndjson-map 'd.id = d.properties.GEOID.slice(2), d' \
  < mt-albers.ndjson \
  > mt-albers-id.ndjson
We need this id to join the geometry with the population estimates.
I was supposed to use curl to download the population estimate: curl 'http://api.census.gov/data/2015/acs5?get=B01003_001E&for=tract:*&in=state:30' -o cb_2017_30_tract_B01003.json.
It created the json file but it was blank. So I went directly to the above url and copied the data into my json file.
The B01003_001E specifies how you want the data. That code specifies the total population estimate. You can do it by gender, race, age, etc,. The for=tract and the in=state specifies that we want data for each census tract in Montana. The resulting JSON is an array and we need it in a NDJSON stream. Use ndjson-cat (to remove the newlines), ndjson-split (to separate the arrau into multiple lines) and ndjson-map (to reformat each line as an object). Run the following: ndjson-cat cb_2017_30_tract_B01003.json \
  | ndjson-split 'd.slice(1)' \
  | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}' \
  > cb_2017_30_tract_B01003.ndjson
Now join the population data to the geometry using ndjson-join: ndjson-join 'd.id' \
  mt-albers-id.ndjson \
  cb_2017_30_tract_B01003.ndjson \
  > mt-albers-join.ndjson
Each line in the resulting NDJSON stream is a two-element array. The first element (d[0]) is from mt-albers-id.ndjson: a GeoJSON feature representing a census tract polygon. The second element (d[1]) is from cb_2017_30_tract_B01003.ndjson: an object representing the population estimate for the same census tract.
Next compute the population density using ndjson-map and remove the additional properties we do not need: ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' \
  < mt-albers-join.ndjson \
  > mt-albers-density.ndjson
The population density is computed as the population estimate B01003 divided by the land area ALAND. The constant 2589975.2356 = 1609.34 squared converts the land area from square meters to square miles.
Next convert back to GeoJSON: ndjson-reduce \
  < mt-albers-density.ndjson \
  | ndjson-map '{type: "FeatureCollection", features: d}' \
  > mt-albers-density.json
The map is now upside down when viewed in mapshaper because +y is treated as up. The coordinate system used by Canvas and SVG treat +y as down. 
We can use d3-geo-projection to quickly generate an SVG choropleth. Install D3: npm install -g d3
Next use ndjson-map requiring D3 via -r d3 and defining a fill property using a sequential scale with the Viridis color scheme: ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000])(d.properties.density), d)' \
  < mt-albers-density.ndjson \
  > mt-albers-color.ndjson
Then convert the newline-delimited GeoJSON to SVG using geo2svg: geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  < mt-albers-color.ndjson \
  > mt-albers-color.svg
Next shrink geometry without loss of detail for web viewing. We will switch to JSON dialect designed for efficient transport: TopoJSON. TopoJSON files are often 80% smaller than GeoJSON.
TopoJSON represents lines and polygons as sequences of acrs rather than sequences of coordinates. Contiguous polygons have shared borders whose coordinates are duplicated in GeoJSON. By representing lines and polygons as sequences of arcs, repeating an arc does not require repeating coordinates.
Install TopoJSON: npm install -g topojson
Use geo2topo to convert to TopoJSON: geo2topo -n \
  tracts=mt-albers-density.ndjson \
  > mt-tracts-topo.json
Now toposimplify: toposimplify -p 1 -f \
  < mt-tracts-topo.json \
  > mt-simple-topo.json
The -p 1 argument tells toposimplify to use a planar area threshold of one square pixel. The -f says to remove small detached rings--little islands, but not contiguous tracts--further reducing the output size. Probably did not need this since Montana is land locked.
Next topoquantize and delta-encode to reduce size further: topoquantize 1e5 \
  < mt-simple-topo.json \
  > mt-quantized-topo.json
Next overlay county borders on the map. Census tracts compose hierarchically into counties so I can derive county geometry using TopoJSON (topomerge): topomerge -k 'd.id.slice(0, 3)' counties=tracts \
  < mt-quantized-topo.json \
  > mt-merge-topo.json 
The -k argument defines a key expression that topomerge will evaluate to group features from the tracts objects before merging. The first three digits of the census tract id represent the state specific part of the county FIPS (Montana is 30) code, so the census tracts for each county will be merged, resulting in county polygons.
We only want the internal borders (the ones separating counties). A filter expression (-f) is evaluated for each arc, given the arc's adjacent polygons a and b. A and b are the same on exterior arcs. So command: topomerge --mesh -f 'a !== b' counties=counties \
  < mt-merge-topo.json \
  > mt-topo.json
Applied a color encoding of the population data to the geometry previously prepared. Applied ndjson-map to the feature collection as a single entity. Allows computing the maximum density of any tract.
Used topo2geo to extract the simplified tracts from the topology, pipe to ndjson-map to assign the fill property for each tract, pipe to ndjson-split to break the collection into features and lastly pipe to geo2svg: topo2geo tracts=- \
  < mt-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0, 4000]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt-tracts-color.svg
Transition from low-density to high-density happens too quickly. Used a non-linear transform to distribute colors more equitably. Implement an exponential transform by passing the square root of the population density to the color scale instead: topo2geo tracts=- \
  < mt-topo.json \
  | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0, 100]), d.features.forEach(f => f.properties.fill = z(Math.sqrt(f.properties.density))), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt-tracts-sqrt.svg
Comparison also completed a log transform: topo2geo tracts=- \
  < mt-topo.json \
  | ndjson-map -r d3 'z = d3.scaleLog().domain(d3.extent(d.features.filter(f => f.properties.density), f => f.properties.density)).interpolate(() => d3.interpolateViridis), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt-tracts-log.svg
To visualize the density's p-quantile instead of it absolute value use: topo2geo tracts=- \
  < mt-topo.json \
  | ndjson-map -r d3 'z = d3.scaleQuantile().domain(d.features.map(f => f.properties.density)).range(d3.quantize(d3.interpolateViridis, 256)), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt-tracts-quantile.svg
This shows variation but it is harder to reason the color variation. So applied a discrete color scheme instead of a continuous one. Install: npm install -g d3-scale-chromatic
Then implement a threshold scale with the OrRd color scheme (used file - see line 67): topo2geo tracts=- \
  < mt-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt-tracts-threshold.svg
Then added county borders to the choropleth: (topo2geo tracts=- \
    < mt-topo.json \
    | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' \
    | ndjson-split 'd.features'; \
topo2geo counties=- \
    < mt-topo.json \
    | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d')\
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > mt.svg
Lastly added a legend to the code in mt.svg on line 277 inside the closing svg tag. Only did this in the mt.svg in my react app.
Open using: http-server -p 8008 &