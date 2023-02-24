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

  let visible_overlays = []

  let year_layers = []

  let overlay_layers = [
    {
      // 'year': year,
      'display_name': 'U-owned properties',
      'source_id': 'u-properties',
      'credit': 'University of Minnesota Real Estate Office',
      'geojson_path': 'https://hsph-urban-renewal.s3.us-east-2.amazonaws.com/overlays/u_realestate_tracts.geojson',
      'visible': false,
      'popup_attrs': [
        'OBJECTID',
        'TRACT_NUM',
        'TRACT_NAME',
        'COUNTY_PID',
        'PRIMARY_ADDRESS',
        'CITY',
        'COUNTY',
        'STATE',
        'ZIP',
        'PIN',
        'LEGAL_DESCRIPTION',
        'REPORTING_CAMPUS',
        'TENURE',
        'LAND_CLASS',
        'ACQUISITION_DATE',
        'ACQUISITION_COST',
        'PREVIOUS_OWNER',
        'ACQUISITION_TYPE',
        'DEED_TYPE',
        'ABSTRACT_DOC',
        'ABSTRACT_BOOK_TYPE_PAGE',
        'ABSTRACT_RECORDING_DATE',
        'TORRENS_DOC',
        'TORRENS_CERTIFICATE',
        'TORRENS_BOOK_TYPE_PAGE',
        'TORRENS_RECORDING_DATE',
      ]
    }
  ]

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

  const addTileLayer = function (map, which_map, layer_config) {
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

  const addGeoJSONLayer = function (map, which_map, layer_config) {
    map.addSource(`${layer_config.source_id}-geojson`, {
      type: 'geojson',
      // Use a URL for the value for the `data` property.
      data: layer_config.geojson_path
    });

    var visibility = 'none';
    if (layer_config['visible'] === true) {
      visibility = 'visible';
      visible_overlays.push(layer_config.source_id);
    }

    map.addLayer({
      'id': layer_config.source_id,
      'type': 'fill',
      'source': `${layer_config.source_id}-geojson`,
      'paint': {
        'fill-color': '#0080ff', // blue color fill
        'fill-opacity': 0.5
      },
      'layout': {
        // Make the layer not visible by default.
        'visibility': visibility,
      },
    });

    if (layer_config['popup_attrs']) {
      map.on('click', layer_config.source_id, (e) => {

        var html = popup_overlay_attr_mapper(e.features[0].properties, layer_config['popup_attrs'])

        new mapboxgl.Popup({maxWidth: '350px'})
          .setLngLat(e.lngLat)
          .setHTML(html)
          .addTo(map);
      });
    }
  };

  const popup_overlay_attr_mapper = function (properties, attr_list) {
    var html = '';

    attr_list.forEach((key) => {
      html += `${key}: ${properties[key]}<br/>`
    })

    return html
  }

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

  const overlayToggle = function (e) {
    // Generally based on https://docs.mapbox.com/mapbox-gl-js/example/toggle-layers/
    var clickedOverlay = this.id;
    var already_visible = visible_overlays.includes(this.id)
    var overlay_obj = overlay_layers.filter(overlay => overlay.short_id == clickedOverlay)[0]

    var maps = Object.keys(map_sides);
    maps.forEach((which_map) => {
      if (already_visible === true) {
        // Toggle to off, hide layers and remove from visible list
        map_sides[which_map].setLayoutProperty(clickedOverlay, "visibility", "none");
        visible_overlays = visible_overlays.filter(overlay => overlay != clickedOverlay)
      } else {
        // Toggle to on, show layers and add to visible list
        map_sides[which_map].setLayoutProperty(clickedOverlay, "visibility", "visible");
        if (!visible_overlays.includes(clickedOverlay)) {
          visible_overlays = [...visible_overlays, clickedOverlay]
        }
      }
    })
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

      // tile layers
      year_layers.forEach(layer => {
        addTileLayer(map, which_map, layer)
      });

      // overlays
      overlay_layers.forEach(layer => {
        addGeoJSONLayer(map, which_map, layer)
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

  #layer-menu {
    position: absolute;
    right: 10px;
    top: 50px;
    z-index: 1000;
  }

  #layer-menu button {
    font-size: 13px;
    color: #404040;
    display: block;
    width: 150px;
    margin: 0;
    padding: 0;
    padding: 5px;
    text-decoration: none;
    border-bottom: 1px solid rgba(0, 0, 0, 0.25);
    text-align: center;
    cursor: pointer;
    border-radius: 5px;
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
  <nav id="layer-menu">
    {#each overlay_layers as layer}
      <button id={layer.source_id} class:active="{visible_overlays.includes(layer.source_id)}" on:click={overlayToggle}>{layer.display_name}</button>
    {/each}
  </nav>
  <div id="map-canvas-left" class="map"></div>
  <div id="map-canvas-right" class="map"></div>
</div>
