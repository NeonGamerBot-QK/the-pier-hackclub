<script lang="ts">
    import { Readable } from "svelte/store";
    import { ChatMessageContent } from "../../../Connection/ChatConnection";

    export let content: Readable<ChatMessageContent>;
</script>

{#if $content.url !== undefined}
    <!-- svelte-ignore a11y-media-has-caption -->
    <video controls class="tw-w-full tw-block tw-rounded">
        <source src={$content.url} />
    </video>
{/if}
