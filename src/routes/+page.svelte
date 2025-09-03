<script lang="ts">
  import Balatro from "./Balatro.svelte";

  let contrast = $state(3.5);
  let pixelFilter = $state(1000);
</script>

<div style="width: 100%; height: 100%;">
  <Balatro {contrast} {pixelFilter} />

  <input type="range" bind:value={contrast} min={0} max={10} step={0.1} />

  <input type="range" bind:value={pixelFilter} min={1} max={1000} step={1} />
</div>
