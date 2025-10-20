<script>
  import Scene from '$lib/components/Scene.svelte';
</script>

<svelte:head>
  <title>Interactive Solar System</title>
  <meta name="description" content="Explore our solar system interactively" />
</svelte:head>

<main>
  <Scene />
  
  <div class="header">
    <h1>Interactive Solar System</h1>
    <p>Click on any planet to learn more</p>
  </div>
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

  .header {
    position: fixed;
    top: 20px;
    left: 20px;
    color: white;
    z-index: 10;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.8);
  }

  .header h1 {
    margin: 0;
    font-size: 2rem;
    background: linear-gradient(45deg, #fdb813, #ffc849);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .header p {
    margin: 5px 0 0 0;
    opacity: 0.8;
  }
</style>