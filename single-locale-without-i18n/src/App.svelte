<script lang="ts">
  import { onMount, untrack, type Component } from 'svelte';
  import Example01 from './Example01.svelte';
  import Example02 from './Example02.svelte';
  import Example03 from './Example03.svelte';

  const examples = [
    { id: '01', name: '1- Basic Grid / 2 Grids' },
    { id: '02', name: '2- Localization (i18n)' },
    { id: '03', name: '3- Master/Detail Grids' },
  ];

  // eager loading exceptions for the first example and the two infinite-scroll demos,
  // which have historically been sensitive to async route loading in Cypress.
  const eagerExampleComponents: Record<string, Component> = {
    '01': Example01,
    '02': Example02,
    '03': Example03,
  };
  const lazyExampleModules = import.meta.glob<{ default: Component }>([
    './Example*.svelte',
    '!./Example01.svelte',
  ]);

  let current = $state(window.location.hash.match(/^#\/example(\d+)$/)?.[1] ?? (window.location.hash === '#/home' ? 'home' : '01'));
  let menuOpen = $state(false);
  let CurrentExample: Component | undefined = $state.raw();
  let isLoadingExample = $state(false);
  let failedToLoadExample = $state(false);
  let exampleLoadSequence = 0;
  const selected = $derived(examples.find((example) => example.id === current));

  async function loadExampleComponent(exampleId: string) {
    const loadSequence = ++exampleLoadSequence;
    CurrentExample = undefined;
    failedToLoadExample = false;

    if (exampleId === 'home') {
      isLoadingExample = false;
      return;
    }

    const eagerComponent = eagerExampleComponents[exampleId];
    const moduleLoader = eagerComponent ? async () => ({ default: eagerComponent }) : lazyExampleModules[`./Example${exampleId}.svelte`];

    if (!moduleLoader) {
      isLoadingExample = false;
      return;
    }

    isLoadingExample = true;
    try {
      const exampleModule = await moduleLoader();
      if (loadSequence === exampleLoadSequence) {
        CurrentExample = exampleModule.default;
      }
    } catch (error) {
      console.error(`Failed to load Svelte example ${exampleId}`, error);
      if (loadSequence === exampleLoadSequence) {
        failedToLoadExample = true;
      }
    } finally {
      if (loadSequence === exampleLoadSequence) {
        isLoadingExample = false;
      }
    }
  }

  $effect(() => {
    const exampleId = current;
    untrack(() => void loadExampleComponent(exampleId));
  });

  function syncRoute() {
    const match = window.location.hash.match(/^#\/example(\d+)$/);
    current = match?.[1] ?? (window.location.hash === '#/home' ? 'home' : '01');
  }

  onMount(() => {
    syncRoute();
    window.addEventListener('hashchange', syncRoute);
    return () => window.removeEventListener('hashchange', syncRoute);
  });
</script>

<nav class="navbar navbar-expand-md navbar-dark bg-dark fixed-top">
  <a class="navbar-brand ms-2" href="https://github.com/ghiscoding/slickgrid-universal/tree/master/frameworks/slickgrid-svelte">
    <i class="mdi mdi-github" aria-hidden="true"></i><span class="ms-2">Slickgrid-Svelte</span>
  </a>
  <div class="github-button-container">
    <a href="https://github.com/ghiscoding/slickgrid-universal">
      <img src="https://img.shields.io/github/stars/ghiscoding/slickgrid-universal?style=social" alt="GitHub stars" />
    </a>
  </div>
  <button
    class="navbar-toggler"
    type="button"
    aria-label="Toggle navigation"
    aria-expanded={menuOpen}
    aria-controls="navbarContent"
    onclick={() => (menuOpen = !menuOpen)}
  >
    <span class="navbar-toggler-icon"></span>
  </button>
  <div id="navbarContent" class:show={menuOpen} class="navbar-collapse collapse justify-content-end me-2">
    <ul class="navbar-nav">
      <li class="nav-item"><a class="nav-link" href="#/home" onclick={() => (menuOpen = false)}>Home</a></li>
      <li class="nav-item">
        <a class="nav-link" href="https://github.com/ghiscoding/slickgrid-universal/tree/master/frameworks/slickgrid-svelte#readme">
          Documentation
        </a>
      </li>
    </ul>
  </div>
</nav>

<div class="panel-wm">
  <aside id="panel-left" class="panel-wm-left" aria-label="Examples">
    <ul class="nav nav-pills nav-stacked">
      <li class="nav-item fw-bold nav-docs">
        <a class="nav-link" href="https://github.com/ghiscoding/slickgrid-universal/tree/master/frameworks/slickgrid-svelte#readme">
          📘 Documentation
        </a>
      </li>
      {#each examples as example (example.id)}
        <li class="nav-item">
          <a
            class="nav-link"
            class:active={current === example.id}
            aria-current={current === example.id ? 'page' : undefined}
            href={'#/example' + example.id}
          >
            {example.name}
          </a>
        </li>
      {/each}
    </ul>
  </aside>
  <section class="panel-wm-content">
    <div id="demo-container" class:grid8={current === '08'}>
      {#if CurrentExample}
        <CurrentExample />
      {:else if isLoadingExample}
        <div>Loading...</div>
      {:else if failedToLoadExample}
        <p>Failed to load this example. Try refreshing the page.</p>
      {:else}
        <h2>{selected?.name ?? 'Example not found'}</h2>
        <p>{selected ? 'This example has not been implemented in Svelte yet.' : 'Choose an example from the sidebar.'}</p>
      {/if}
    </div>
  </section>
</div>
