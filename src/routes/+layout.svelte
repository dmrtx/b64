<script lang="ts">
	import { pwaInfo } from "virtual:pwa-info";
	import { onMount } from "svelte";

	let { children } = $props();

	onMount(async () => {
		if (pwaInfo) {
			const { registerSW } = await import("virtual:pwa-register");
			registerSW({
				immediate: true,
				onRegistered(r) {
					console.log("SW Registered: " + r);
				},
				onRegisterError(error) {
					console.log("SW registration error", error);
				},
			});
		}
	});
</script>

<svelte:head>
	<link rel="icon" href="/icon.svg" />
	{@html pwaInfo ? pwaInfo.webManifest.linkTag : ""}
</svelte:head>

{@render children()}
