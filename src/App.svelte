<script lang="ts">
  import svelteLogo from './assets/svelte.svg'
  import viteLogo from '/vite.svg'
  import Counter from './lib/Counter.svelte'
  import './app.css';

  import { MapLibre, DefaultMarker, Popup, FillLayer, GeoJSON, hoverStateFilter, LineLayer } from 'svelte-maplibre';

  import { createClient } from '@supabase/supabase-js'
    import { onMount } from 'svelte';

  let checked = true;
  let distance = 500;
  let radius_schools = false;
  let radius_betting_places = false;
  let show_distance_between_points = false;

  let data_circles;
  let data_lines;

  const supabase = createClient('https://jwgzmsqtacrmbpnzmxky.supabase.co', 
 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Imp3Z3ptc3F0YWNybWJwbnpteGt5Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3MDg0MzYwMjYsImV4cCI6MjAyNDAxMjAyNn0.E77waVrgzNb1R0fnV9QyjS-iri9j53r_JB2OYNrYEbk')

  async function fetch_db(){
    const { data } = await supabase.rpc("all_records_geojson")
    console.log(data)
    return {
      places: data ?? [],
    }
  };

  async function fetch_schools(){
    const { data } = await supabase.rpc("all_records_school_geojson")
    console.log(data)
    return {
      places: data ?? [],
    }
  };

  async function fetch_count_betting_places() {
    const { count, error } = await supabase.from('betting_places').select('*', { count: 'exact', head: true })
    return count
  }

    async function fetch_count_schools() {
    const { count, error } = await supabase.from('export_osm').select('*', { count: 'exact', head: true })
    return count
  }

  function concat_json(data1, data2) {
    data1['places']['features'] = data1['places']['features'].concat(data2['places']['features']); 
    return data1['places'];
  }

  async function fetch_circles() {
    console.log("CIRCLES EXECUTED")
    const { data } = await supabase.rpc("circle_radius_all", {radius: distance})
    console.log(data)
    return {
      places: data ?? [],
    }
  }

  async function fetch_lines() {
    console.log("LINES EXECUTED")
    const { data } = await supabase.rpc("line_geometry_between_places", {radius: distance})
    console.log(data)
    return {
      places: data ?? [],
    }
  }

 $: distance, fetch_lines().then(data => {
    data_lines = data['places']

    fetch_circles().then(data => {
      console.log("CHANGED STATE")
      data_circles = data['places']
    })
  })

  onMount(async () => {
    await fetch_circles();
  })
</script>

<main>
  <h1>Кладиници и Училишта</h1>

  {#await fetch_count_betting_places()}
  {:then count}
  <div class="stats shadow">
  <div class="stat">
    <div class="stat-title">Вкупно кладилници</div>
    <div class="stat-value">{count}</div>
    <div class="stat-desc">Вкупно заведени кладилници</div>
  </div>
</div>
{:catch}
<h3>Error fetching data</h3>
{/await}

  {#await fetch_count_schools()}
  {:then count}
  <div class="stats shadow">
  <div class="stat">
    <div class="stat-title">Вкупно училишта</div>
    <div class="stat-value">{count}</div>
    <div class="stat-desc">Вкупно мапирани училишта</div>
  </div>
</div>
{:catch}
<h3>Error fetching data</h3>
{/await}

  <div class="form-control">

    <h3>Радиус: {distance} метри</h3>
    <input type="range" min="0" max="2000" bind:value={distance} class="range range-primary" />
    
    <label class="label cursor-pointer">
      <span class="label-text">Кругчиња училишта?</span> 
      <input type="checkbox" bind:checked={radius_schools} class="checkbox checkbox-primary" />
    </label>

    <label class="label cursor-pointer">
      <span class="label-text">Кругчиња кладилници?</span> 
      <input type="checkbox" bind:checked={radius_betting_places} class="checkbox checkbox-primary" />
    </label>

    <label class="label cursor-pointer">
      <span class="label-text">Покажи растојанија помеѓу местата?</span> 
      <input type="checkbox" bind:checked={show_distance_between_points} class="checkbox checkbox-primary" />
    </label>
 
  </div>

{#if distance}
  {#await fetch_db()}
  <p>waiting</p>
  {:then data}
  <MapLibre 
    center={[21.7289728,41.7146641]}
    zoom={7}
    class="map"
    standardControls
    style="https://basemaps.cartocdn.com/gl/positron-gl-style/style.json" >
  {#each data['places']['features'] as resp}
  <DefaultMarker lngLat={resp['geometry']['coordinates']} >
    <Popup offset={[0, -10]}>
      <div class="text-lg font-bold" style="color:black">{JSON.stringify(resp)}</div>
    </Popup>
  </DefaultMarker>
  {/each}
  {#if checked}
    {#await fetch_schools()}
      <p>Downloading...</p>
    {:then data_schools}
    {#if radius_betting_places || radius_schools}
      <GeoJSON id="circles" data={data_circles}>
        {#if radius_betting_places}
        <FillLayer
          paint={{
            'fill-color': hoverStateFilter("#32DE8A", "#32DE8A"),
            'fill-opacity': 0.5,
          }}
          beforeLayerType="symbol"
          manageHoverState
          filter={["==", "type", "betting_place"]}
        >
        <Popup openOn={"click"} closeOnClickInside let:features>
          {@const props = features?.[0]?.properties}
            <p style="color:black"><b>{JSON.stringify(props)}</b></p>
        </Popup>
        </FillLayer>
        <LineLayer
          layout={{ 'line-cap': 'round', 'line-join': 'round' }}
          paint={{ 'line-color': "#32DE8A", 'line-width': 3 }}
          beforeLayerType="symbol"
          filter={["==", "type", "betting_place"]}
        />
        {/if}
        {#if radius_schools}
        <h3>{JSON.stringify(data_circles)}</h3>
        <FillLayer
          paint={{
            'fill-color': hoverStateFilter("#3F88C5", "#44BBA4"),
            'fill-opacity': 0.5,
          }}
          beforeLayerType="symbol"
          manageHoverState
          filter={["==", "type", "school"]}
        >
        <Popup openOn={"click"} closeOnClickInside let:features>
          {@const props = features?.[0]?.properties}
            <p style="color:black"><b>{JSON.stringify(props)}</b></p>
        </Popup>
        </FillLayer>
        <LineLayer
          layout={{ 'line-cap': 'round', 'line-join': 'round' }}
          paint={{ 'line-color': "#3F88C5", 'line-width': 3 }}
          beforeLayerType="symbol"
          filter={["==", "type", "school"]}
        />
        {/if}
      </GeoJSON>
      {/if}

      {#if show_distance_between_points}
      <h3>{JSON.stringify(data_lines)}</h3>
        <GeoJSON data={data_lines} id="lines-distance">
          <LineLayer
            layout={{ 'line-cap': 'round', 'line-join': 'round' }}
            paint={{ 'line-color': "#3F88C5", 'line-width': 3 }}
            beforeLayerType="symbol"
          >
          <Popup openOn={"click"} closeOnClickInside let:features>
            {@const props = features?.[0]?.properties}
              <p style="color:black"><b>{JSON.stringify(props)}</b></p>
          </Popup>
        </LineLayer>
        </GeoJSON>
      {/if}


    <GeoJSON id="info" data={concat_json(data, data_schools)}>
      <FillLayer
          paint={{
            'fill-color': hoverStateFilter("#C7FFED", "#C7FFED"),
            'fill-opacity': 0.5,
          }}
          beforeLayerType="symbol"
          manageHoverState
        />
    </GeoJSON>
    {:catch error}
      <p>{error.message}</p>
    {/await}
  {/if}
  </MapLibre>
  {:catch error}
  <p>{error.message}</p>
  {/await}
{/if}

</main>
<style>
  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: filter 300ms;
  }
  .logo:hover {
    filter: drop-shadow(0 0 2em #646cffaa);
  }
  .logo.svelte:hover {
    filter: drop-shadow(0 0 2em #ff3e00aa);
  }
  .read-the-docs {
    color: #888;
  }

  :global(.map) {
    height: 500px;
    width: 100em;
  }

  select {
    margin-bottom: 50px;
    margin-top: 50px;
  }

  .marker-school > svg > g > path {
    fill: red;
}

.stats {
  margin-top: 50px;
}
</style>
