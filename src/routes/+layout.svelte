<script lang="ts">
	let { children } = $props();
</script>

<svelte:head>
    <title>版本管理面板</title>
</svelte:head>

{@render children()}
