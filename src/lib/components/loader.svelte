<script>
	import { SyncLoader } from 'svelte-loading-spinners';

	export let size = 60;
	export let center = true;
</script>

<div class={center ? 'center' : ''}>
	<SyncLoader {size} />
</div>
