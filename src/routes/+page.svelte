<script>
  import { onMount } from "svelte";
  import mapboxgl from "mapbox-gl";
  import MapboxGeocoder from "@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.min.js";
  import "mapbox-gl/dist/mapbox-gl.css";
  import '@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.css';

  let map;
  mapboxgl.accessToken =
    "pk.eyJ1IjoibWFwcGluZ3ByZWp1ZGljZSIsImEiOiJjbDJ6MW92eDIwMGdzM2NxenQ0MGk3b3BhIn0.JPNTwpmUJahPgt6LUlX9fg";

  let visible_years = ['aerial-1940']
  let year_layers = []

  const avail_years = [1940, 1957, 1964, 1966, 1969, 1988]
  // Tile extent: 465791.5084,499545.2019,4962112.6447,4990268.9920 [EPSG:26915]
  avail_years.forEach(year => {
    let credit = "Adapted from Hennepin County GIS"
    if (year == 1966) {
      credit = "Adapted from Hennepin County GIS"
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

  const addLayer = function (map, layer_config) {
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

    if (!visible_years.includes(layer_config.source_id)) {
      map.setLayoutProperty(layer_config.source_id, "visibility", "none");
    }
  };

  const yearToggle = function (e) {
    // Generally based on https://docs.mapbox.com/mapbox-gl-js/example/toggle-layers/
    var clickedYear = this.id;
    var already_visible = visible_years.includes(this.id)
    var year_obj = year_layers.filter(year => year.source_id == clickedYear)[0]
    // console.log(current_map_type)

    if (already_visible === true) {
      // Toggle to off, hide layers and remove from visible list
      map.setLayoutProperty(year_obj.source_id, "visibility", "none");
      visible_years = visible_years.filter(year => year != clickedYear)
    } else {
      // Toggle to on, show layers and add to visible list
      map.setLayoutProperty(year_obj.source_id, "visibility", "visible");
      if (!visible_years.includes(clickedYear)) {
        visible_years = [...visible_years, clickedYear]
      }
    }
  };

  onMount(() => {

    map = new mapboxgl.Map({
        container: 'map-canvas', // container ID
        style: 'mapbox://styles/mapbox/streets-v12', // style URL
        center: [-93.23, 44.9655], // starting position [lng, lat]
        zoom: 14 // starting zoom
    });

    map.addControl(new MapboxGeocoder({
        accessToken: mapboxgl.accessToken,
        mapboxgl: mapboxgl,
        collapsed: true
    }), 'top-left');
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
        addLayer(map, layer)
      });
    });

  });

</script>

<style>
  #map-container {
    position: relative;
  }

  #map-canvas {
    position: absolute;
    top: 50px;
    width: 1200px;
    height: 700px;
  }

  #layer-menu {
    background: #fff;
    border-radius: 3px;
    border: 1px solid rgba(0, 0, 0, 0.4);
    /* font-family: "Open Sans", sans-serif; */
    font-weight: 700;
    /* text-transform: uppercase; */
  }

  #layer-menu {
    position: absolute;
    z-index: 1;
    top: 0px;
    left: 0px;
  }

  #layer-menu button {
    font-size: 13px;
    color: #404040;
    display: inline-block;
    width: 150px;
    margin: 0;
    padding: 0;
    padding: 10px;
    text-decoration: none;
    border-bottom: 1px solid rgba(0, 0, 0, 0.25);
    text-align: center;
    cursor: pointer;
  }

  #layer-menu button:hover {
    color: #000;
    cursor: pointer;
    /* background: #3074a4; */
  }

  #layer-menu button.active {
    background-color: #fec057;
    color: #FFF;
    border-color: #CCC;
    /* cursor: default; */
  }
</style>

<h1>HSPH comparison map</h1>

<div id="map-container">
  <nav id="layer-menu">
    {#each year_layers as layer}
      <button id={layer.source_id} class:active="{visible_years.includes(layer.source_id)}" on:click={yearToggle}>{layer.year}</button>
    {/each}
  </nav>
  <div id="map-canvas" />
</div>
