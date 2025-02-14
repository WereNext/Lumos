<script lang="ts">
  import "../app.css";
  import { fade } from 'svelte/transition';
  import { darkMode } from '$lib/stores/theme';
  import TokenDisplay from '$lib/components/TokenDisplay.svelte';
  import ThemeDebug from '$lib/components/ThemeDebug.svelte';
  import { page } from '$app/stores';
  import type { TransitionConfig } from 'svelte/transition';
  import { writable } from 'svelte/store';

  interface PageData {
    title?: string;
  }

  const TRANSITION_DURATION = 200;
  const THEME_DEBOUNCE = 200;

  let transitioning = false;
  let error: Error | null = null;
  let togglePromise: Promise<void> | null = null;
  
  // Create a store for the illuminated state
  const isIlluminated = writable(false);

  function handleError(e: ErrorEvent) {
    error = e.error;
    console.error('Layout error:', e.error);
  }

  async function toggleTheme(): Promise<void> {
    if (transitioning || togglePromise) return;
    transitioning = true;
    
    try {
      togglePromise = new Promise(resolve => {
        darkMode.toggle();
        setTimeout(() => {
          transitioning = false;
          togglePromise = null;
          resolve();
        }, THEME_DEBOUNCE);
      });
      await togglePromise;
    } catch (e) {
      console.error('Theme toggle failed:', e);
      transitioning = false;
      togglePromise = null;
    }
  }

  function toggleIllumination() {
    isIlluminated.update(state => !state);
  }

  const fadeConfig: TransitionConfig = {
    duration: TRANSITION_DURATION
  };

  $: title = ($page.data as PageData).title 
    ? `${($page.data as PageData).title} | Lumos`
    : 'Lumos - Share Your Insights';
</script>

<svelte:window on:error={handleError} />

<!-- Move svelte:head outside of any blocks -->
<svelte:head>
  <title>{title}</title>
  <meta name="description" content="Lumos - A decentralized platform for sharing and discovering insights">
  <meta property="og:title" content={title}>
  <meta property="og:description" content="Share and discover insights on Lumos">
  <meta property="og:type" content="website">
  <meta property="og:image" content="/og-image.png">
  <meta name="twitter:card" content="summary_large_image">
  <link rel="icon" type="image/png" href="%sveltekit.assets%/favicon.png">
  <link rel="preconnect" href="https://fonts.googleapis.com">
</svelte:head>

{#if error}
  <div class="error-boundary">
    <p>Something went wrong. Please refresh the page.</p>
  </div>
{:else}
  <div class="app" class:dark={$darkMode}>
    <div class="min-h-screen bg-gradient-light dark:bg-gradient-dark transition-colors duration-200">
      <!-- Enhanced Header -->
      <header class="sticky top-0 z-50 backdrop-blur-lg bg-white/60 dark:bg-black/60 border-b border-surface-200 dark:border-surface-700">
        <div class="container mx-auto px-4 py-3">
          <nav class="flex items-center justify-between">
            <!-- Left side: Logo and nav links -->
            <div class="flex items-center space-x-8">
              <a 
                href="/" 
                class="flex items-center gap-2 text-2xl font-bold transition-all duration-300
                  {$isIlluminated ? 
                    'text-yellow-400 hover:text-yellow-500' : 
                    'text-primary-600 hover:text-primary-700'}"
                on:click|preventDefault={(e) => {
                  toggleIllumination();
                  // Still navigate home if that's where we're not already
                  if (window.location.pathname !== '/') {
                    window.location.href = '/';
                  }
                }}
              >
                <svg 
                  xmlns="http://www.w3.org/2000/svg" 
                  class="h-7 w-7 transition-all duration-300 transform
                    {$isIlluminated ? 'filter-glow text-yellow-400' : ''}" 
                  viewBox="0 0 24 24" 
                  fill={$isIlluminated ? 'currentColor' : 'none'}
                  stroke="currentColor"
                >
                  <path 
                    stroke-linecap="round" 
                    stroke-linejoin="round" 
                    stroke-width="2" 
                    d="M13 10V3L4 14h7v7l9-11h-7z"
                  />
                </svg>
                <span class={$isIlluminated ? 'text-glow' : ''}>Lumos</span>
              </a>
            </div>

            <!-- Center: Navigation links -->
            <div class="hidden md:flex items-center space-x-1">
              <a href="/" class="px-4 py-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all">Home</a>
              <a href="/popular" class="px-4 py-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all">Popular</a>
              <a href="/all" class="px-4 py-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all">All</a>
            </div>

            <!-- Right side: Search and actions -->
            <div class="flex items-center space-x-4">
              <!-- Search Bar -->
              <div class="relative w-64">
                <input 
                  type="search" 
                  placeholder="Search posts..."
                  class="w-full px-4 py-2 
                    bg-surface-100 dark:bg-black/30 
                    rounded-lg border border-transparent 
                    focus:border-primary-500 
                    focus:bg-white dark:focus:bg-black/40 
                    transition-all"
                >
              </div>

              <!-- Action buttons in a single row -->
              <div class="flex items-center space-x-2">
                <TokenDisplay />
                
                <!-- New Post Button -->
                <button class="p-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4" />
                  </svg>
                </button>

                <!-- Theme Toggle -->
                <button 
                  class="p-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all"
                  on:click={toggleTheme}
                >
                  {#if $darkMode}
                    <!-- Sun icon -->
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z" />
                    </svg>
                  {:else}
                    <!-- Moon icon -->
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
                    </svg>
                  {/if}
                </button>

                <!-- Notifications -->
                <button class="p-2 rounded-lg text-surface-600 dark:text-surface-300 hover:bg-surface-100 dark:hover:bg-black/40 transition-all">
                  <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
                  </svg>
                </button>
              </div>
            </div>
          </nav>
        </div>
      </header>

      <!-- Main Content with Animation -->
      <main class="container mx-auto px-4 py-6" in:fade={{ duration: 200 }}>
        <slot />
      </main>
    </div>
  </div>

  {#if import.meta.env.DEV}
    <ThemeDebug />
  {/if}
{/if}

<style>
  .app {
    display: contents;
  }

  /* Single transition definition for all elements */
  :global(body) {
    transition: background-color 200ms ease-out;
  }
  
  :global(.theme-transition) {
    transition-property: background-color, border-color, color;
    transition-timing-function: ease-out;
    transition-duration: 200ms;
  }

  .filter-glow {
    filter: drop-shadow(0 0 8px rgb(250, 204, 21));
    animation: pulse 2s ease-in-out infinite;
  }
  
  .text-glow {
    text-shadow: 0 0 8px rgb(250, 204, 21),
                 0 0 12px rgb(250, 204, 21, 0.8);
    animation: text-pulse 2s ease-in-out infinite;
  }

  @keyframes pulse {
    0% {
      filter: drop-shadow(0 0 8px rgb(250, 204, 21));
    }
    50% {
      filter: drop-shadow(0 0 12px rgb(250, 204, 21));
    }
    100% {
      filter: drop-shadow(0 0 8px rgb(250, 204, 21));
    }
  }

  @keyframes text-pulse {
    0% {
      text-shadow: 0 0 8px rgb(250, 204, 21),
                   0 0 12px rgb(250, 204, 21, 0.8);
    }
    50% {
      text-shadow: 0 0 12px rgb(250, 204, 21),
                   0 0 16px rgb(250, 204, 21, 0.8);
    }
    100% {
      text-shadow: 0 0 8px rgb(250, 204, 21),
                   0 0 12px rgb(250, 204, 21, 0.8);
    }
  }
</style> 