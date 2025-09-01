<script lang="ts">
  import * as Tooltip from "$lib/components/ui/tooltip/index.js";
  import ClubRail from "$lib/components/layout/ClubRail.svelte";
  import type { ClubSummary } from "$lib/types/club.js";
  let { children } = $props();

  const clubs: ClubSummary[] = [
    { id: "homebrew", name: "Homebrew Guild", accentColor: "#fbff86", unreadCount: 3 },
    { id: "synthwave", name: "Synthwave Cafe", accentColor: "#905ea9" },
    { id: "devs", name: "Dev Collective", accentColor: "#30e1b9", mentionCount: 1 },
    { id: "pixels", name: "Pixel Artists", accentColor: "#fbb954" },
    { id: "gamers", name: "Night Gamers", accentColor: "#4d9be6" },
    { id: "plants", name: "Plant Parents", accentColor: "#91db69" },
  ];

  let selectedClubId: string | null = $state(clubs[0]?.id ?? null);

  function handleSelect(payload: { id: string | null }) {
    selectedClubId = payload.id;
  }
</script>

<Tooltip.Provider>
  <div class="flex min-h-svh">
    <ClubRail {clubs} {selectedClubId} select={handleSelect} class="shrink-0" />
    <div class="flex-1 min-w-0">
      {@render children?.()}
    </div>
  </div>
</Tooltip.Provider>
