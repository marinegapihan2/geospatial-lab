<h1>Bikewatching</h1>
<p>Building an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
mapboxgl.accessToken = "pk.eyJ1IjoibWFyaW5lZ2FwaWhhbiIsImEiOiJjbTk0c3h6bmgwd2hzMnJweTRlaHlqN3lyIn0.zQlSkV4IXf490212y0j28Q";

import { onMount } from "svelte";

import * as d3 from "d3";
let stations = [];
let map;
let mapViewChanged = 0;
$: map?.on("move", evt => mapViewChanged++);

let trips=[];
let departures;
let arrivals;


onMount(async () => {
	stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
	trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv");


	departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
	arrivals=d3.rollup(trips, v => v.length, d => d.end_station_id);

stations = stations.map(station => {
	let id = station.Number;
	station.arrivals = arrivals.get(id) ?? 0;
	station.departures= departures.get(id)?? 0;
	station.totalTraffic = station.arrivals + station.departures;
	return station;
	});

});

function getCoords (station) {
	let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
	let {x, y} = map.project(point);
	return {cx: x, cy: y};
}

$: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
	.range([0, 25]);

async function initMap() {
	map = new mapboxgl.Map({
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

<div id="map">
	<svg style="pointer-events:none;">
		{#key mapViewChanged}
		{#each stations as station}
		<circle
			{ ...getCoords(station) }
			r={radiusScale(station.totalTraffic)}
			style="fill: steelblue; fill-opacity:0.6; stroke:white; pointer-events:auto;">
			<title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
		</circle>

	{/each}
	{/key}
	</svg>
</div>


<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
}
#map svg {
	position: absolute;
	z-index: 1;
	width: 100%;
	height: 100%;
	pointer-events: none;
}

circle{
	pointer-events: auto;
}
</style>
