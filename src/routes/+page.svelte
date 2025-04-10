<header>
    <h1>🚲 BikeWatch</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>


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

const departuresByMinute = Array.from({length: 1440}, () => []);
const arrivalsByMinute = Array.from({length: 1440}, () => []);

let stationFlow = d3.scaleQuantize()
	.domain([0, 1])
	.range([0, 0.5, 1]);


onMount(async () => {
	stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
	trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
	for (let trip of trips) {
		trip.started_at = new Date(trip.started_at)
		trip.ended_at = new Date(trip.ended_at)
		let startedMinutes = minutesSinceMidnight(trip.started_at);
departuresByMinute[startedMinutes].push(trip);
		let endedMinutes = minutesSinceMidnight(trip.ended_at);
arrivalsByMinute[startedMinutes].push(trip);
	}
	return trips;
});

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

function minutesSinceMidnight (date) {
	return date.getHours() * 60 + date.getMinutes();
}

function filterByMinute (tripsByMinute, minute) {
	let minMinute = (minute - 60 + 1440) % 1440;
	let maxMinute = (minute + 60) % 1440;

	if (minMinute > maxMinute) {
		let beforeMidnight = tripsByMinute.slice(minMinute);
		let afterMidnight = tripsByMinute.slice(0, maxMinute);
		return beforeMidnight.concat(afterMidnight).flat();
	}
	else {
		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}
}

$: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
	let startedMinutes = minutesSinceMidnight(trip.started_at);
	let endedMinutes = minutesSinceMidnight(trip.ended_at);
	return Math.abs(startedMinutes - timeFilter) <= 60
	       || Math.abs(endedMinutes - timeFilter) <= 60;
});

$: filteredDepartures = timeFilter === -1
    ? d3.rollup(trips, v => v.length, d => d.start_station_id)
    : d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);

$: filteredArrivals = timeFilter === -1
    ? d3.rollup(trips, v => v.length, d => d.end_station_id)
    : d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);

$: filteredStations = stations.map(station => {
	const id = station.Number;
	const arr = filteredArrivals.get(id) ?? 0;
	const dep = filteredDepartures.get(id) ?? 0;
	return {
		...station,
		arrivals: arr,
		departures: dep,
		totalTraffic: arr + dep
	};
});

$: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
  .range(timeFilter === -1 ? [0, 25] : [3, 50]);


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

let timeFilter = -1;
$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});


const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
const profile = 'cycling';
const minutes = [5, 10, 15, 20];
const contourColors = [
	"03045e",
	"0077b6",
	"00b4d8",
	"90e0ef"
]
let isochrone = null;

async function getIso(lon, lat) {
	const base = `${urlBase}${profile}/${lon},${lat}`;
	const params = new URLSearchParams({
		contours_minutes: minutes.join(','),
		contours_colors: contourColors.join(','),
		polygons: 'true',
		access_token: mapboxgl.accessToken
	});
	const url = `${base}?${params.toString()}`;

	const query = await fetch(url, { method: 'GET' });
	isochrone = await query.json();

}
let selectedStation = null;

function geoJSONPolygonToPath(feature) {
	const path = d3.path();
	const rings = feature.geometry.coordinates;

	for (const ring of rings) {
		for (let i = 0; i < ring.length; i++) {
			const [lng, lat] = ring[i];
			const { x, y } = map.project([lng, lat]);
			if (i === 0) path.moveTo(x, y);
			else path.lineTo(x, y);
		}
		path.closePath();
	}
	return path.toString();
}

$: if (selectedStation) {
	getIso(+selectedStation.Long, +selectedStation.Lat);
} else {
	isochrone = null;
}

</script>

<div id="map">
    <svg>
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                            d={geoJSONPolygonToPath(feature)}
                            fill={feature.properties.fillColor}
                            fill-opacity="0.2"
                            stroke="#000000"
                            stroke-opacity="0.5"
                            stroke-width="1"
						>
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
	<circle
		{...getCoords(station)}
		r={radiusScale(station.totalTraffic)}
		class={station?.Number === selectedStation?.Number ? "selected" : ""}
		on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}
		style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }">
		<title>{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)</title>
	  </circle>

	{/each}
	{/key}
	</svg>
</div>

<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
}
@import url("$lib/global.css");

:root {
  --color-departures: steelblue;
  --color-arrivals: darkorange;
}

#map {
  flex: 1;
}

#map svg {
  position: absolute;
  z-index: 1;
  width: 100%;
  height: 100%;
  pointer-events: all; /* Allow pointer events on the entire SVG */
}

#map svg path{
	pointer-events: auto;
}

#map svg circle {
	opacity: 1;  /* Default faded state */
  stroke: white;
}

#map svg circle:not(.selected) {
  opacity: 0.6;  /* Dim non-selected circles */
  fill-opacity: 0.6;
}

#map svg circle .selected {
  opacity: 1; /* Full opacity for selected circle */
  stroke: black;  /* Optional: Make the selected circle stand out */
}


#map svg circle :has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
		fill-opacity: 0.6;
	}

#map circle {
  --color: color-mix(
    in oklch,
    var(--color-departures) calc(100% * var(--departure-ratio)),
    var(--color-arrivals)
  );
  fill: var(--color);
}

.legend > div {
  --color: color-mix(
    in oklch,
    var(--color-departures) calc(100% * var(--departure-ratio)),
    var(--color-arrivals)
  );
  background-color: var(--color);
  flex: 1;
  padding: 0.4em 1em;
  color: white;
  text-align: center;
}

.legend {
  display: flex;
  margin-block: 1rem;
  gap: 1px;
  font-size: 0.9rem;
  font-weight: bold;
  border: 1px solid #ccc;
  border-radius: 4px;
  overflow: hidden;
}

header {
  display: flex;
  gap: 1em;
  align-items: baseline;
}

label {
  margin-left: auto;
}

time {
  display: block;
}

em {
  display: block;
  color: gray;
  font-style: italic;
}

</style>
