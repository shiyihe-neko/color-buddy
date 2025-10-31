<script lang="ts">
  import { onMount } from "svelte";
  import { get } from "svelte/store";
  import configStore from "../stores/config-store";
  import colorStore from "../stores/color-store";
  import { Color } from "color-buddy-palette";
  import type { Palette } from "color-buddy-palette";
  import Tooltip from "../components/Tooltip.svelte";
  import { buttonStyle } from "../lib/styles";
  import QuestionIcon from "virtual:icons/fa6-solid/circle-question";
  import ChevDown from "virtual:icons/fa6-solid/chevron-down";

  // ⌨️ Keyboard shortcuts
  const isMac = navigator.userAgent.indexOf("Mac OS X") !== -1;
  const metaKey = isMac ? "⌘" : "ctrl";
  const altKey = isMac ? "option" : "alt";
  const shortCuts = [
    { name: "Undo", shortcut: `${metaKey}+z` },
    { name: "Redo", shortcut: `${metaKey}+y` },
    { name: "Select All", shortcut: `${metaKey}+a` },
    { name: "Delete Selection", shortcut: "delete" },
    { name: "Copy", shortcut: `${metaKey}+c` },
    { name: "Paste", shortcut: `${metaKey}+v` },
    { name: "Move in x-y", shortcut: "arrow keys" },
    { name: "Move in z", shortcut: `${altKey}+up/down keys` },
  ];

  // 🧠 强制默认使用 Anthropic
  onMount(() => {
    if (get(configStore).engine !== "anthropic") {
      configStore.setEngine("anthropic");
    }
  });

  // 🎨 导入调色板功能
  function importPals() {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = ".json";
    input.onchange = (e: any) => {
      const file = e.target.files[0];
      const reader = new FileReader();
      reader.onload = (e) => {
        const text = (e.target as FileReader).result as string;
        const palettes = JSON.parse(text);
        if (Array.isArray(palettes)) {
          const pals: Palette[] = palettes
            .filter((x) => {
              const { colors, background, name, colorSpace, type } = x;
              return (
                Array.isArray(colors) &&
                colors.every((c: any) => typeof c === "string") &&
                typeof background === "string" &&
                typeof name === "string" &&
                typeof colorSpace === "string" &&
                typeof type === "string"
              );
            })
            .map((x) => {
              const { colors, background, name, colorSpace, type } = x;
              return {
                background: Color.colorFromString(background, colorSpace),
                colorSpace,
                colors: colors.map((c: string) =>
                  Color.colorFromString(c, colorSpace)
                ),
                colorSemantics: colors.map(() => ({
                  size: undefined,
                  markType: undefined,
                  tags: [],
                })),
                evalConfig: {},
                name,
                type,
                tags: [],
                folder: "",
              };
            });
          colorStore.setPalettes(pals);
        }
      };
      reader.readAsText(file);
    };
    input.click();
  }
</script>

<Tooltip positionAlongRightEdge={true}>
  <button
    class="text-white flex items-center mr-10"
    slot="target"
    let:toggle
    on:click={toggle}
  >
    <QuestionIcon />
    <ChevDown class="ml-2" />
  </button>

  <div slot="content" class="w-96" let:onClick>
    <div class="font-bold">About</div>
    <div class="text-sm my-2">
      Color Buddy is a tool for building color palettes. Learn more at the <a class="underline text-cyan-800" href="https://color-buddy-docs.netlify.app/" target="_blank">documentation</a>. Feedback via GitHub issues or email is welcome.
    </div>

    <div class="font-bold">What is saved and where?</div>
    <div class="text-sm my-2">
      Palettes are saved in your browser's local storage and not shared. They are unavailable on other browsers or devices. A small amount of non-identifiable usage data is collected to help improve the application.
    </div>

    <div class="font-bold">Tools</div>
    <div class="flex">
      <button
        class={buttonStyle}
        on:click={() => {
          configStore.setTour(true);
          onClick();
        }}
      >
        Show Tour
      </button>
    </div>

    <div class="font-bold mt-4">Shortcuts</div>
    <div class="text-sm">
      {#each shortCuts as { name, shortcut }}
        <div class="flex justify-between">
          <div class="mr-4">{name}</div>
          <div>{shortcut}</div>
        </div>
      {/each}
    </div>
  </div>
</Tooltip>