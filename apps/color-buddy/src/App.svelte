<script lang="ts">
  import { onMount } from "svelte";

  import colorStore from "./stores/color-store";
  import focusStore from "./stores/focus-store";
  import configStore from "./stores/config-store";
  import lintStore from "./stores/lint-store";

  import { logEvent } from "./lib/api-calls";

  let showPrompt = true;

  onMount(() => {
    logEvent("start-up", {}, $configStore.userName);

    if (sessionStorage.getItem("cb_seen_prompt") === "1") showPrompt = false;

    const url = new URL(window.location.href);
    const pal = deserializePaletteForUrl(window.location.href);
    if (pal) {
      colorStore.createNewPal(pal);
      url.searchParams.delete("colors");
      url.searchParams.delete("bg");
      url.searchParams.delete("name");
      url.searchParams.delete("space");
      window.history.replaceState({}, "", url.toString());
    }
  });

  function acknowledge() {
    showPrompt = false;
    sessionStorage.setItem("cb_seen_prompt", "1");
    const focusTarget =
      document.getElementById("kb-focus-target") ||
      document.getElementById("app-root") ||
      document.body;
    focusTarget?.focus?.({ preventScroll: true });
  }

  // make sure no focused colors are out of bounds
  $: focusedColors = $focusStore.focusedColors;
  $: currentPal = $colorStore.palettes[$colorStore.currentPal];
  $: palPresent = !!($colorStore.palettes.length > 0 && currentPal);
  $: {
    if (focusedColors.some((x) => x > currentPal.colors.length - 1)) {
      focusStore.clearColors();
    }
    if ($colorStore.palettes.length > 0 && !currentPal) {
      colorStore.startUsingPal(0);
    }
  }

  import Nav from "./components/Nav.svelte";
  import NewPal from "./controls/NewPal.svelte";

  import LeftPanel from "./content-modules/LeftPanel.svelte";
  import Examples from "./example/Examples.svelte";
  import Eval from "./linting/Eval.svelte";
  import KeyboardHooks from "./components/KeyboardHooks.svelte";
  import ComparePal from "./content-modules/ComparePal.svelte";
  import Manage from "./content-modules/Manage.svelte";
  import MainColumn from "./content-modules/MainColumn.svelte";
  import TourProvider from "./content-modules/TourProvider.svelte";
  import Config from "./controls/Config.svelte";
  import Title from "./controls/Title.svelte";
  import SharePal from "./components/SharePal.svelte";

  import { lint } from "./lib/api-calls";
  import { debounce } from "vega";

  import { buttonStyle } from "./lib/styles";
  import { deserializePaletteForUrl } from "./lib/utils";

  $: route = $configStore.route;
  $: evalRoute = $configStore.evalDisplayMode;
  const bindStr = "!!";
  let updateSearchDebounced = debounce(10, (x: [any, string]) => {
    const [pal, ignoreString] = x;
    if ((route !== "eval" || evalRoute !== "check-customization") && pal) {
      lintStore.setLoadState("loading");
      const outPal = {
        ...pal,
        evalConfig: {
          ...pal.evalConfig,
          globallyIgnoredLints: ignoreString.split(bindStr),
        },
      };
      lint(outPal, true).then((res): void => {
        lintStore.postCurrentChecks(res);
      });
    }
  });
  $: globalString = $colorStore.globallyIgnoredLints.join(bindStr);
  $: globalString, updateSearchDebounced([currentPal, globalString]);

  let innerWidth = window.innerWidth;
  let leftPanelWidth = 320;
  $: columnWidth = (innerWidth - leftPanelWidth) / 2;
  const padding = 40;
  const zWidth = 110;
  $: scatterSize = Math.max(Math.min(columnWidth - zWidth - padding, 420), 300);

  const currentPalTabs = ["examples", "compare", "eval"];
  $: numPassing = $lintStore.currentChecks.filter(
    (x) => x.kind === "success" && !x.passes
  ).length;
</script>

<div
  id="kb-focus-target"
  tabindex="0"
  style="position:fixed;left:-9999px;top:auto;width:1px;height:1px;overflow:hidden;"
></div>

<header class="flex w-full bg-stone-800 justify-between min-h-12">
  <div class="flex" id="top-controls">
    <div class="text-4xl font-bold text-white px-2 py-1 flex">
      <img src="logo.png" alt="logo" class="h-10 mr-2" />
      <div class="">Color Buddy</div>
    </div>
    <div>
      <div class="flex h-12 items-center">
        <NewPal />
        <Manage />
        <div>
          <button
            class={`${buttonStyle} ml-2 mr-1`}
            on:click={() => colorStore.undo()}
          >
            Undo
          </button>
        </div>
        <div>
          <button
            class={`${buttonStyle} mr-2`}
            on:click={() => colorStore.redo()}
          >
            Redo
          </button>
        </div>
        <SharePal />
      </div>
    </div>
  </div>
  <div class="flex justify-between items-center">
    <a
      href="https://github.com/mcnuttandrew/color-buddy"
      class="text-sm"
      target="_blank"
    >
      <img src={"./github-mark-white.png"} alt="github" class="h-5 mr-4" />
    </a>
    <Config />
  </div>
</header>

<main class="flex h-full" id="app-root" tabindex="-1">
  <div class="flex flex-col">
    <Title />
    <div class="flex h-full">
      <LeftPanel />
      <div
        class="h-full flex flex-col grow main-content border-b border-l border-stone-200"
      >
        {#if palPresent}
          <MainColumn {scatterSize} />
        {:else}
          <div
            class="flex-grow flex justify-center items-center"
            style={`width: ${columnWidth}px`}
          >
            <div class="text-2xl max-w-md text-center">
              No palettes present, click "New" in the upper left to create a new
              one, or "Browse" to pick from existing ones.
            </div>
          </div>
        {/if}
      </div>
    </div>
  </div>

  <div
    class="flex flex-col w-full border-b border-l border-stone-200 h-full"
    id="right-col"
  >
    <div class="flex bg-stone-200 w-full border-b border-l border-stone-200">
      <Nav
        className="mt-1"
        tabs={currentPalTabs}
        isTabSelected={(x) => $configStore.route === x}
        selectTab={(x) => {
          // @ts-ignore
          configStore.setRoute(x);
        }}
        formatter={(x) =>
          x === "eval" ? "Evaluation" : x === "compare" ? "Comparison" : "Examples"
        }
      >
        <div slot="menu" let:tab>
          {#if tab === "eval" && numPassing > 0}
            <svg width={`${18}px`} height={`${18}px`} class="ml-1">
              <circle
                r={9}
                cx={9}
                cy={9}
                fill={"rgb(185 28 28 / var(--tw-bg-opacity))"}
              />
              <text
                x={9}
                y={13.5}
                font-size={13}
                fill="white"
                text-anchor="middle"
                font-weight="normal"
              >
                {numPassing}
              </text>
            </svg>
          {/if}
        </div>
      </Nav>
    </div>
    <div class=" h-full">
      {#if palPresent && $configStore.route === "examples"}
        <Examples />
      {:else if palPresent && $configStore.route === "compare"}
        <ComparePal {scatterSize} />
      {:else if palPresent && $configStore.route === "eval"}
        <Eval />
      {/if}
    </div>
  </div>
</main>

<KeyboardHooks />
<svelte:window bind:innerWidth />
{#if $configStore.tour}
  <TourProvider />
{/if}

{#if showPrompt}
  <div class="cb-toast cb-toast-center">
    <strong>Start using Color Buddy</strong>
    <p style="margin: 8px 0 16px;">Click “Okay” to begin.</p>
    <button on:click={acknowledge}>Okay</button>
  </div>
{/if}

<style>
  .main-content {
    min-width: 0;
  }

  .cb-toast {
    background: #fff;
    border: 1px solid #e2e8f0;
    border-radius: 16px;
    padding: 40px 60px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
    z-index: 9999;
    min-width: 360px;
    max-width: 90%;
    text-align: center;
    font-size: 1.2rem;
    line-height: 1.6;
    color: #111;
  }

  .cb-toast-center {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }

  .cb-toast button {
    padding: 10px 24px;
    border-radius: 8px;
    border: none;
    background: #2b8a3e;
    color: #fff;
    cursor: pointer;
    font-size: 1rem;
    transition: all 0.2s ease;
  }

  .cb-toast button:hover {
    background: #256f33;
    transform: scale(1.05);
  }
</style>
