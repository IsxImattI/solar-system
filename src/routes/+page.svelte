<script>
  import Scene from '$lib/components/Scene.svelte';
</script>

<svelte:head>
  <title>Interactive Solar System</title>
  <meta name="description" content="Explore our solar system interactively" />
</svelte:head>

<main>
  <Scene />
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  main {
    position: relative;
    width: 100vw;
    height: 100vh;
  }
</style>