<h1>Bikewatching</h1>
<p>Building an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
mapboxgl.accessToken = "pk.eyJ1IjoibWFyaW5lZ2FwaWhhbiIsImEiOiJjbTk0c3h6bmgwd2hzMnJweTRlaHlqN3lyIn0.zQlSkV4IXf490212y0j28Q";

import { onMount } from "svelte";

async function initMap() {
	let map = new mapboxgl.Map({
    container: "map",
      style: "mapbox://styles/marinegapihan/cm94ts66z000u01s51e9h25mm",
      zoom: 12,
      center: [-71.0589, 42.3601],
	});

    await new Promise(resolve => map.on("load", resolve));

	map.addSource("boston_route", {
		type: "geojson",
		data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });

    map.addLayer({
	id: "bike-lanes", // A name for our layer (up to you)
	type: "line", // one of the supported layer types, e.g. line, circle, etc.
	source: "boston_route", // The id we specified in `addSource()`
	paint: {
	   "line-color":"green",
       "line-width": 2,
       "line-opacity": 0.4,
	},
});

map.addSource("cambridge_route", {
		type: "geojson",
		data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });
map.addLayer({
	id: "cambridge-bike-lanes", // A name for our layer (up to you)
	type: "line", // one of the supported layer types, e.g. line, circle, etc.
	source: "cambridge_route", // The id we specified in `addSource()`
	paint: {
	   "line-color":"green",
       "line-width": 2,
       "line-opacity": 0.4,
	},
});

}

onMount(() => {
	initMap();
});

</script>

<div id="map" />

<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
}

</style>
