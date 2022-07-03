<script>
	import { fly } from 'svelte/transition';

	export let notification = 'notification';
	export let type = 'info';
	export let show = true;
</script>

{#if show}
	<div transition:fly={{ y: -200 }} class={type + ' container'}>
		<p class="notification">{notification}</p>
	</div>
{/if}

<style>
	.container {
		font-family: 'Quicksand', sans-serif;
		font-weight: 600;
		color: #fff;
		border: 0;
		padding: 12px;
		padding-right: 24px;
		border-radius: 4px;
		min-width: 200px;
		max-width: 300px;

		position: absolute;
		top: 12px;
		right: 16px;
	}
	.info {
		background-color: var(--primary-color);
	}
	.error {
		background-color: var(--error-color);
	}
	.notification {
		margin: 0;
	}
</style>
