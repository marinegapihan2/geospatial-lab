<h1>Bikewatching</h1>
<p>Building an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
mapboxgl.accessToken = "pk.eyJ1IjoibWFyaW5lZ2FwaWhhbiIsImEiOiJjbTk0c3h6bmgwd2hzMnJweTRlaHlqN3lyIn0.zQlSkV4IXf490212y0j28Q";

import { onMount } from "svelte";

onMount(() => {
	let map = new mapboxgl.Map({
    container: "map",
      style: "mapbox://styles/marinegapihan/cm94ts66z000u01s51e9h25mm",
      zoom: 12,
      center: [-71.0589, 42.3601],
	});
})

</script>

<div id="map" />

<style>
    @import url("$lib/global.css");
    #map {
	flex: 1;
}

</style>
