<script>
    import { Canvas } from '@threlte/core';
    import Scene from './Scene.svelte';
</script>

<div class="full-screen-container">
  <Canvas>
    <Scene />
  </Canvas>
</div>

<style>
  :global(body, html) {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
  }

  .full-screen-container {
    width: 100%;
    height: 100%;
    position: fixed;
    top: 0;
    left: 0;
  }
</style>