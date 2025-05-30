<script>
  import world from "$data/110m.json";
  import Glow from "$components/Glow.svelte";
  import data from "$data/data.json";
  import Tooltip from "$components/Tooltip.svelte";

  import * as topojson from "topojson-client";
  import { geoOrthographic, geoPath } from "d3-geo";
  import { scaleLinear } from "d3-scale";
  import { max } from "d3-array";
  import { timer } from "d3-timer";
  import { spring } from "svelte/motion";

  let countries = topojson.feature(world, world.objects.countries).features;
  let borders = topojson.mesh(
    world,
    world.objects.countries,
    (a, b) => a !== b,
  );

  countries.forEach((country) => {
    const metadata = data?.find((d) => d.id === country.id);
    if (metadata) {
      country.total = metadata.total;
      country.pais = metadata.pais;
    }
  });

  let width = 400;
  $: height = width;

  $: projection = geoOrthographic()
    .scale(width / 2)
    .rotate([$rotation, -10, 20])
    .translate([width / 2, height / 2]);

  $: path = geoPath(projection);

  const colorScale = scaleLinear()
    .domain([0, max(data, (d) => d.total)])
    .range(["#ffd4cc", "#e63315"]);

  let rotation = spring(0, { stiffness: 0.08, damping: 0.4 });
  let direction = 1;
  let minRotation = -20;
  let maxRotation = 100;
  let speed = 0.25;

  const t = timer(() => {
    if (dragging || tooltipData) return;

    rotation.update((r) => {
      let next = r + speed * direction;
      if (next <= minRotation || next >= maxRotation) {
        direction *= -1;
        next = Math.max(minRotation, Math.min(maxRotation, next));
      }
      return next;
    });
  });

  let globe;

  import { onMount } from "svelte";
  import { drag } from "d3-drag";
  import { select } from "d3-selection";
  import { geoCentroid } from "d3-geo";

  const DRAG_SENSITIVITY = 0.1;
  let dragging = false;

  onMount(() => {
    const element = select(globe);
    element.call(
      drag()
        .on("drag", (event) => {
          dragging = true;
          rotation.update((r) => {
            let next = r + event.dx * DRAG_SENSITIVITY;
            next = Math.max(minRotation, Math.min(maxRotation, next));
            return next;
          });
        })
        .on("end", () => {
          dragging = false;

          if ($rotation <= minRotation) direction = 1;
          else if ($rotation >= maxRotation) direction = -1;
        }),
    );
  });

  let tooltipData;

  $: if (tooltipData) {
    const center = geoCentroid(tooltipData);
    $rotation = -center[0];
  }

    window.addEventListener("DOMContentLoaded", (event) => {
    function updateIframeHeight() {
      const el = document.documentElement;
      const rect = el.getBoundingClientRect();
      const styles = window.getComputedStyle(el);
      const margin =
        parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
      const height = Math.ceil(rect.height + margin);

      window.parent.postMessage(
        {
          type: "resize-iframe",
          value: height,
        },
        "*",
      );
    }
    updateIframeHeight();

    if (window.ResizeObserver) {
      new ResizeObserver(() => {
        updateIframeHeight();
      }).observe(document.documentElement);
    } else {
      window.addEventListener("load", updateIframeHeight);
      window.addEventListener("resize", updateIframeHeight);
    }

    window.addEventListener(
      "message",
      (event) => {
        if (event.data.type === "request-resize") {
          updateIframeHeight();
        }
      },
      false,
    );
  });
</script>

<div class="chart-container" bind:clientWidth={width}>
  <svg {width} {height} bind:this={globe} class:dragging>
    <Glow />
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <circle
      cx={width / 2}
      cy={height / 2}
      r={width / 2}
      fill={"#ffe9e7"}
      filter="url(#glow)"
      on:click={() => {
        tooltipData = null;
      }}
    />
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    {#each countries as country}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <path
        d={path(country)}
  fill={
    country.properties.name === "Spain"
      ? "grey"
      : country?.total > 0
        ? colorScale(country.total)
        : "white"
  }        stroke="none"
        cursor={country?.total > 0 ? "pointer" : "default"}
        on:click={() => {
          if (country.total > 0) tooltipData = country;
        }}
      />
    {/each}
    <path d={path(borders)} fill="none" stroke="white" stroke-width="0.25" />
    {#if Tooltip}
      <path
        d={path(tooltipData)}
        fill="transparent"
        stroke="white"
        stroke-width="2"
      />
    {/if}
  </svg>
  <Tooltip data={tooltipData} />
</div>

<style>
  .chart-container {
    max-width: 468px;
    margin: 0 auto;
  }

  svg {
    overflow: visible;
  }

  .dragging {
    cursor: grabbing;
  }

  h1,
  h2 {
    text-align: center;
  }

  h1 {
    font-size: 1.75rem;
    font-weight: 800;
    margin-bottom: 0.35rem;
  }

  h2 {
    font-size: 1.25rem;
    font-weight: 200;
    margin-bottom: 1rem;
  }
</style>
