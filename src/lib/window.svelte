<script lang="ts">
	import type { Snippet } from "svelte";

	interface Props {
		class?: string;
		children: Snippet;
	}

	const props: Props = $props();
</script>

<div class={"p-1 bg-(--window-bg) text-center rounded-2xl " + props.class}>{@render props.children()}</div>
