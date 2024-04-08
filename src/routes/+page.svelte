<header>
    <h1>🚴🏻‍♀️ Bikewatching</h1>
    <label>Filter by time:
        <input type=range min="-1" max="1440" bind:value={timeFilter}/>
        <time>{ timeFilterLabel }</time>
    </label>
    <em>(any time)</em>
</header>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each filteredStations as station}
                <circle { ...getCoords(station) } r={ radiusScale(station.totalTraffic) } fill="steelblue" />
            {/each}
        {/key}
    </svg>
</div>


<script>
    import * as d3 from "d3";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1Ijoic29ncmVlbiIsImEiOiJjbHVwc280cm0xOHU0MmxtcnE2ZGdveXF5In0.hbXdn_hdpt1MX0gubJtbLw";
    import { onMount } from "svelte";

    let stations = [];
    let trips = [];
    let map, arrivals, departures;
    let mapViewChanged = 0;
    $: map?.on("move", evt => mapViewChanged++);
    let timeFilter = -1;
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});


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
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    })
    
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic)])
        .range([0, 25]);

    function getCoords (station) {
            let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
            let {x, y} = map.project(point);
            return {cx: x, cy: y};
    }

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

    $: filteredStations = stations.map(station => {
        let id = station.Number;
        station = {...station};
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });
</script>

<style>
    @import url("$lib/global.css");
</style>