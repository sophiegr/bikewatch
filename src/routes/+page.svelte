<h1>🚴🏻‍♀️ Bikewatching</h1>

<script>
    import * as d3 from "d3";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1Ijoic29ncmVlbiIsImEiOiJjbHVwc280cm0xOHU0MmxtcnE2ZGdveXF5In0.hbXdn_hdpt1MX0gubJtbLw";
    import { onMount } from "svelte";

    let stations = [];
    let map;

    onMount(async () => {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            style: "mapbox://styles/mapbox/streets-v12",
            zoom: 12,
        });
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "boston_layer", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "purple",
                "line-width": 3,
                "line-opacity": 0.4,
            },
        });
        map.addLayer({
            id: "cambridge_layer", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "purple",
                "line-width": 3,
                "line-opacity": 0.4,
            },
        });

        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    })

    function getCoords (station) {
            let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
            let {x, y} = map.project(point);
            return {cx: x, cy: y};
    }
</script>

<div id="map">
    <svg>
        {#each stations as station}
            <circle { ...getCoords(station) } r="5" fill="steelblue" />
        {/each}
    </svg>
</div>

<style>
    @import url("$lib/global.css");
</style>