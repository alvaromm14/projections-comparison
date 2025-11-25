<script>
  import { onMount } from "svelte";
  import * as topojson from "topojson-client";
  import { geoPath, geoMercator, geoCentroid, geoBounds } from "d3-geo";
  import worldData from "$data/110m.json";

  const africaCodes = [
    "AGO",
    "BFA",
    "BDI",
    "BEN",
    "BWA",
    "CAF",
    "CIV",
    "CMR",
    "COD",
    "COG",
    "COM",
    "CPV",
    "DJI",
    "DZA",
    "EGY",
    "ERI",
    "ESH",
    "ETH",
    "GAB",
    "GHA",
    "GIN",
    "GMB",
    "GNB",
    "GNQ",
    "KEN",
    "LBR",
    "LBY",
    "LSO",
    "MAR",
    "MDG",
    "MLI",
    "MOZ",
    "MRT",
    "MUS",
    "MWI",
    "NAM",
    "NER",
    "NGA",
    "RWA",
    "SDN",
    "SEN",
    "SLE",
    "SOM",
    "SSD",
    "STP",
    "SWZ",
    "SYC",
    "TCD",
    "TGO",
    "TUN",
    "TZA",
    "UGA",
    "ZAF",
    "ZMB",
    "ZWE",
    "SOL",
  ];

  let width = 900;
  const height = 400;
  const mapW = width / 2;
  const mapH = mapW < 500 ? height - 70 : height;

  let africaFeature = null;
  let greenlandFeature = null;
  let projAfrica = null;
  let projGreenland = null;
  let pathAfrica = null;
  let pathGreenland = null;
  let africaBBoxArea = 1;
  let greenlandBBoxArea = 1;
  let africaCentroidPixel = [mapW / 2, mapH / 2];
  let greenlandCentroidPixel = [mapW / 2, mapH / 2];
  let slider = 1;
  const sliderMin = 0.2;
  const sliderMax = 1;
  let message = "";

  onMount(() => {
    const world = topojson.feature(worldData, worldData.objects.countries);
    const features = world.features || [];

    africaFeature = {
      type: "FeatureCollection",
      features: features.filter(
        (f) => f.properties?.a3 && africaCodes.includes(f.properties.a3),
      ),
    };

    greenlandFeature = features.find((f) => f.properties?.a3 === "GRL");

    const africaBounds = geoBounds(africaFeature);
    const africaWidth = africaBounds[1][0] - africaBounds[0][0];
    const africaHeight = africaBounds[1][1] - africaBounds[0][1];
    const scaleAfrica =
      mapW > 500
        ? 30 * Math.min(mapW / africaWidth, mapH / africaHeight)
        : 40 * Math.min(mapW / africaWidth, mapH / africaHeight);

    const africaCentroidGeo = geoCentroid(africaFeature);

    projAfrica = geoMercator()
      .scale(scaleAfrica)
      .center(africaCentroidGeo)
      .translate(mapW > 500 ? [mapW / 2, mapH / 2.7] : [mapW / 2, mapH / 2.1]);

    const greenCentroidGeo = geoCentroid(greenlandFeature);
    projGreenland = geoMercator()
      .scale(scaleAfrica)
      .center(greenCentroidGeo)
      .translate(
        mapW > 500 ? [mapW / 2.2, mapH / 1.85] : [mapW / 2, mapH / 1.6],
      );

    pathAfrica = geoPath().projection(projAfrica);
    pathGreenland = geoPath().projection(projGreenland);

    const aBoundsPx = pathAfrica.bounds(africaFeature);
    africaBBoxArea =
      (aBoundsPx[1][0] - aBoundsPx[0][0]) * (aBoundsPx[1][1] - aBoundsPx[0][1]);
    africaCentroidPixel = pathAfrica.centroid(africaFeature);

    const gBoundsPx = pathGreenland.bounds(greenlandFeature);
    greenlandBBoxArea =
      (gBoundsPx[1][0] - gBoundsPx[0][0]) * (gBoundsPx[1][1] - gBoundsPx[0][1]);
    greenlandCentroidPixel = pathGreenland.centroid(greenlandFeature);
  });

  $: currentVisualRatio =
    africaBBoxArea / (slider * slider * greenlandBBoxArea);

  function resetMessage() {
    message = "";
  }

  function checkRatio() {
    const targetRatio = 14;
    const diff = currentVisualRatio - targetRatio;
    const percent = (currentVisualRatio ** 0.5 / targetRatio ** 0.5) * 100;

    if (Math.abs(diff) < 3) {
      message = `✅ ¡Correcto! El tamaño de África en realidad multiplica por 14 el de Groenlandia.`;
    } else if (Math.abs(diff) < 5) {
      message = `⚠️ ¡Estás muy cerca (${100 - percent.toFixed()}%)! Ajusta un pelín más la escala.`;
    } else {
      message = `⚠️ Estás a un ${100 - percent.toFixed()}% del objetivo. Sigue ajustando la escala.`;
    }
  }
</script>

<div class="wrap">
  <div class="controls">
    <label class="small">Ajusta el tamaño de Groenlandia:</label>

    <div class="controls-row">
      <input
        type="range"
        min={sliderMin}
        max={sliderMax}
        step="0.001"
        bind:value={slider}
        on:input={resetMessage}
      />
      <button class="btn" on:click={checkRatio}>Comprobar</button>
    </div>
  </div>

  <div class="maps">
    <div class="card">
      <strong>África</strong>
      <svg class="map-svg" viewBox={`0 0 ${mapW} ${mapH}`}>
        {#if africaFeature}
          <path d={pathAfrica(africaFeature)} fill="#dae7f2" />
        {/if}
      </svg>
    </div>

    <div class="card">
      <strong>Groenlandia</strong>
      <svg class="map-svg" viewBox={`0 0 ${mapW} ${mapH}`}>
        {#if greenlandFeature}
          <g
            transform={`translate(${greenlandCentroidPixel[0]},${greenlandCentroidPixel[1]})
                  scale(${slider})
                  translate(${-greenlandCentroidPixel[0]},${-greenlandCentroidPixel[1]})`}
          >
            <path d={pathGreenland(greenlandFeature)} fill="#5ca7e6" />
          </g>
        {/if}
      </svg>
    </div>
  </div>

  {#if message}
    <div class="message">{message}</div>
  {/if}
</div>

<style>
  .wrap {
    width: 100%;
    max-width: 1000px;
    margin: auto;
  }
  .controls {
    display: flex;
    gap: 12px;
    align-items: center;
    flex-wrap: wrap;
    margin: 0 auto 12px auto;
    max-width: 752px;
  }

  .controls label {
    min-width: 110px;
  }

  /* fila interna para slider + botón */
  .controls-row {
    display: flex;
    gap: 8px;
    flex: 1 1 auto;
    align-items: center;
  }

  .controls-row input[type="range"] {
    flex: 1 1 auto;
    height: 5px;
  }
  .btn {
    padding: 6px 10px;
    border-radius: 6px;
    border: none;
    background: #0078d4;
    color: white;
    cursor: pointer;
  }

  .maps {
    display: flex;
    gap: 12px;
    align-items: start;
    flex-wrap: wrap;
  }

  .map-svg path,
  .map-svg g > path {
    transform: translateY(-10px); /* Ajuste fino */
    transform-box: fill-box;
  }

  .card {
    flex: 1 1 48%;
    border: 0.5px solid grey;
    border-radius: 8px;
    padding: 10px;
    box-sizing: border-box;
  }

  .card strong {
    font-weight: bold;
  }

  .map-svg {
    display: block;
    width: 100%;
    height: auto;
  }

  .message {
    margin-top: 15px;
    font-size: 0.9rem;
    text-align: center;
  }

  @media (max-width: 500px) {
    .maps {
      flex-direction: column; /* ya lo tenías */
      align-items: center; /* centra las cards */
    }

    .card {
      flex: 1 1 100%; /* ocupan todo el ancho */
      width: 100%;
    }
  }

  @media (max-width: 500px) {
    .controls {
      flex-direction: column;
      align-items: stretch;
    }

    .controls label {
      width: 100%;
      margin-bottom: -8px;
    }

    .controls-row {
      width: 100%;
    }

    .controls-row button {
      flex: 0 0 auto;
    }
  }
</style>
