<script>
  import { browser } from '$app/environment'
  import { onMount } from "svelte";
  import mapboxgl from "mapbox-gl";
  import MapboxGeocoder from "@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.min.js";
  import "mapbox-gl/dist/mapbox-gl.css";
  import '@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.css';

  // See dynamic import in OnMount
  // import MapboxCompare from "mapbox-gl-compare";

  import 'mapbox-gl-compare/dist/mapbox-gl-compare.css';

  mapboxgl.accessToken =
    "pk.eyJ1IjoibWFwcGluZ3ByZWp1ZGljZSIsImEiOiJjbDJ6MW92eDIwMGdzM2NxenQ0MGk3b3BhIn0.JPNTwpmUJahPgt6LUlX9fg";

  let map_compare;
  let map_sides = {
    left: null,
    right: null
  }

  let visible_year = {
    left: 'aerial-1940',
    right: '2023'
  }

  let year_layers = []

  const avail_years = [1940, 1956, 1957, 1964, 1966, 1969, 1988]
  // Tile extent: 465791.5084,499545.2019,4962112.6447,4990268.9920 [EPSG:26915]
  avail_years.forEach(year => {
    let credit = "Adapted from Hennepin County GIS"
    if (year == 1956 || year == 1966) {
      credit = "Adapted from USpatial"
    }
    year_layers.push({
      'year': year,
      'source_id': `aerial-${year}`,
      'credit': credit,
      'tile_path': `https://hsph-urban-renewal.s3.us-east-2.amazonaws.com/tiles/${year}/{z}/{x}/{y}.png`,
      'minzoom': 10,
      'maxzoom': 17
    })
  });

  const addLayer = function (map, which_map, layer_config) {
    map.addSource(`${layer_config.source_id}-tiles`, {
      'type': 'raster',
      'tiles': [
        layer_config.tile_path
      ],
      'tileSize': 256,
      'tms': false,
      'attribution': layer_config.credit
    });

    map.addLayer({
      'id': layer_config.source_id,
      'type': 'raster',
      'source': `${layer_config.source_id}-tiles`,
      'minzoom': layer_config.minzoom,
      'maxzoom': layer_config.maxzoom,
      },
      // firstSymbolId
      0
    );

    if (!visible_year[which_map].includes(layer_config.source_id)) {
      map.setLayoutProperty(layer_config.source_id, "visibility", "none");
    }
  };

  const yearToggle = function (which_map) {

    var year_obj = year_layers.filter(year => year.source_id == visible_year[which_map])[0]

    if (visible_year[which_map] != 2023) {
      // Don't style, in order to avoid untidy console error
      map_sides[which_map].setLayoutProperty(visible_year[which_map], "visibility", "visible");
    }

    year_layers.forEach(year => {
      if (year.source_id != visible_year[which_map]) {
        map_sides[which_map].setLayoutProperty(year.source_id, "visibility", "none");
      }
    });
  };

  const buildMap = function (map, which_map) {
    map = new mapboxgl.Map({
        container: `map-canvas-${which_map}`, // container ID
        style: 'mapbox://styles/mapbox/satellite-v9', // style URL
        center: [-93.23, 44.9655], // starting position [lng, lat]
        zoom: 14 // starting zoom
    });

    if (which_map == "right") {
      map.addControl(new MapboxGeocoder({
          accessToken: mapboxgl.accessToken,
          mapboxgl: mapboxgl,
          collapsed: true
      }), 'top-right');
    }

    map.addControl(new mapboxgl.NavigationControl(), 'top-left');

    map.on('load', () => {

      const layers = map.getStyle().layers;
      // Find the index of the first symbol layer in the map style.
      let firstSymbolId;
      for (const layer of layers) {
        if (layer.type === 'symbol') {
          firstSymbolId = layer.id;
          break;
        }
      }

      year_layers.forEach(layer => {
        addLayer(map, which_map, layer)
      });
    });

    return map
  }

  // onMount(() => {
  onMount(async () => {
    // This is a hacky fix for issues with MapboxCompare, which, using a standard
    // import, will cause a vite error "window is not defined"
    // Solution here: https://stackoverflow.com/questions/69874742/sveltekit-console-error-window-is-not-defined-when-i-import-library
    const MapboxCompare = (await import('mapbox-gl-compare')).default

    map_sides['left'] = buildMap(map_sides['left'], 'left');
    map_sides['right'] = buildMap(map_sides['right'], 'right');

    map_compare = new MapboxCompare(map_sides['left'], map_sides['right'], '#comparison-container', {
    // Set this to enable comparing two maps by mouse movement:
    // mousemove: true
    });

  });

</script>

<style>

  #control-container {
    width: 100%;
    display: flex;
    justify-content: space-around;
    margin: 20px 0;
  }

  .control-item {
    font-size: 1.2em;
  }

  .control-item select {
    font-size: 1em;
  }

  #comparison-container {
    position: relative;
    top: 0;
    left: 0;
    width: 100%;
    height: 700px;
  }

  .map {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
  }
</style>

<h1>HSPH comparison map</h1>

<div id="control-container">
  <div class="control-item">
    <label>Left side</label>
    <select id="layer-menu-left" bind:value={visible_year['left']} on:change="{() => yearToggle('left')}">
    	{#each year_layers as layer}
    		<option value={layer.source_id}>{layer.year}</option>
    	{/each}
      <option value="2023">2023</option>
    </select>
  </div>

  <div class="control-item">
    <label>Right side</label>
    <select id="layer-menu-right" bind:value={visible_year['right']} on:change="{() => yearToggle('right')}">
    	{#each year_layers as layer}
    		<option value={layer.source_id}>{layer.year}</option>
    	{/each}
      <option value="2023">2023</option>
    </select>
  </div>
</div>

<div id="comparison-container">
  <div id="map-canvas-left" class="map"></div>
  <div id="map-canvas-right" class="map"></div>
</div>
