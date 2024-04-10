<header>
    <h1>Corportate Ownership Rates Over Time in Each Boston Neighborhood</h1>
    <label>Filter by time:
        <input type=range min="-1" max="2024" bind:value={timeFilter}/>
        <time>{ timeFilterLabel }</time>
    </label>
    <em>(any time)</em>
</header>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each neighborhoods as neighborhood}
                <circle { ...getCoords(neighborhood) } r="5" fill="steelblue" />
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
    let neighborhoods = [];
    let stations = [];
    let trips = [];
    let id;
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

        neighborhoods = await d3.csv("/src/data/ownership_over_time.csv")

        


        stations = stations.map(station => {
            let neighborhood = station.Neighborhood;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    })
    
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(neighborhoods, d => d.corp_own_rate)])
        .range([0, 1]);

    function getCoords (station) {
            let point = new mapboxgl.LngLat(+station.Lon, +station.Lat);
            let {x, y} = map.project(point);
            return {cx: x, cy: y};
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