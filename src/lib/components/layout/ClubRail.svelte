<script lang="ts">
  import * as Tooltip from "$lib/components/ui/tooltip/index.js";
  import Avatar from "$lib/components/ui/avatar/avatar.svelte";
  import AvatarImage from "$lib/components/ui/avatar/avatar-image.svelte";
  import AvatarFallback from "$lib/components/ui/avatar/avatar-fallback.svelte";
  import type { ClubSummary } from "$lib/types/club.js";
  import { cn } from "$lib/utils.js";
  import { Home, Plus, Cog } from "@lucide/svelte";

  type Props = {
    clubs?: ClubSummary[];
    selectedClubId?: string | null;
    showHome?: boolean;
    class?: string;
    // Svelte 5: callback props instead of createEventDispatcher
    select?: (payload: { id: string | null }) => void;
    create?: () => void;
    settings?: () => void;
  };

  let {
    clubs = [],
    selectedClubId = null,
    showHome = true,
    class: className,
    select,
    create,
    settings
  }: Props = $props();

  function initials(name: string) {
    const p = name
      .trim()
      .split(/\s+/)
      .map((s) => s[0]?.toUpperCase())
      .filter(Boolean);
    return (p[0] ?? "?") + (p[1] ?? "");
  }
</script>

<nav
  class={cn(
    "bg-sidebar text-sidebar-foreground flex flex-col items-center gap-3 py-3 border-r border-sidebar-border",
    "w-16 min-w-16 h-svh",
    className
  )}
  aria-label="Club navigation"
>
  {#if showHome}
    <Tooltip.Root>
      <Tooltip.Trigger>
        <button
          aria-label="Home"
          class={cn(
            "group relative size-12 rounded-2xl bg-sidebar-accent text-sidebar-foreground grid place-items-center transition-all hover:rounded-xl",
            selectedClubId === null && "ring-2 ring-sidebar-ring"
          )}
          onclick={() => select?.({ id: null })}
        >
          <Home size={22} />
        </button>
      </Tooltip.Trigger>
      <Tooltip.Content side="right">Home</Tooltip.Content>
    </Tooltip.Root>
  {/if}

  <div class="h-px w-10 bg-sidebar-border my-1" aria-hidden="true"></div>

  <div
    class="flex-1 flex flex-col items-center gap-3 overflow-y-auto scrollbar-thin scrollbar-thumb-sidebar-border/60 scrollbar-track-transparent px-1"
  >
    {#each clubs as club (club.id)}
      <Tooltip.Root>
        <Tooltip.Trigger>
          <button
            aria-label={club.name}
            onclick={() => select?.({ id: club.id })}
            class={cn(
              "group relative size-12 rounded-2xl bg-card transition-all hover:rounded-xl overflow-hidden ring-offset-2",
              selectedClubId === club.id ? "ring-2 ring-sidebar-ring" : "ring-0"
            )}
          >
            <Avatar class="size-full">
              {#if club.iconUrl}
                <AvatarImage src={club.iconUrl} alt={club.name} />
              {/if}
              <AvatarFallback
                class="text-xs font-medium"
                style={club.accentColor ? `background:${club.accentColor}; color: var(--sidebar-foreground)` : ""}
              >
                {initials(club.name)}
              </AvatarFallback>
            </Avatar>
            {#if club.unreadCount}
              <span
                class="absolute -top-1 -right-1 min-w-5 h-5 px-1 rounded-full bg-secondary text-[10px] font-semibold text-secondary-foreground grid place-items-center shadow"
              >
                {club.unreadCount > 99 ? "99+" : club.unreadCount}
              </span>
            {/if}
            {#if club.mentionCount}
              <span class="absolute -bottom-1 -right-1 size-3 rounded-full bg-primary ring-2 ring-background"></span>
            {/if}
          </button>
        </Tooltip.Trigger>
        <Tooltip.Content side="right">{club.name}</Tooltip.Content>
      </Tooltip.Root>
    {/each}
  </div>

  <Tooltip.Root>
    <Tooltip.Trigger>
      <button
        aria-label="Create club"
        class="group relative size-12 rounded-2xl bg-muted text-muted-foreground grid place-items-center transition-all hover:rounded-xl"
        onclick={() => create?.()}
      >
        <Plus size={22} />
      </button>
    </Tooltip.Trigger>
    <Tooltip.Content side="right">Create a club</Tooltip.Content>
  </Tooltip.Root>

  <div class="h-px w-10 bg-sidebar-border my-1" aria-hidden="true"></div>

  <Tooltip.Root>
    <Tooltip.Trigger>
      <button
        aria-label="Settings"
        class="group relative size-10 rounded-xl bg-muted text-muted-foreground grid place-items-center transition-all hover:rounded-lg mb-1"
        onclick={() => settings?.()}
      >
        <Cog size={18} />
      </button>
    </Tooltip.Trigger>
    <Tooltip.Content side="right">Settings</Tooltip.Content>
  </Tooltip.Root>
</nav>
