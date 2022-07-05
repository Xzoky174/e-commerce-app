<script>
	import { goto } from '$app/navigation';
	import { supabase } from '$lib/external/supabaseClient';

	async function signOut() {
		await supabase.auth.signOut();
		goto('/?signed_out=true', { replaceState: true });
	}
</script>

<svelte:head>
	<title>Sign Out</title>
</svelte:head>

<div class="container font-raleway">
	<div class="sub-container">
		<h1>Are you sure you want to sign out?</h1>

		<div class="options-container">
			<button on:click={signOut} class="confirm">Confirm</button>
			<a href="/" class="cancel">Cancel</a>
		</div>
	</div>
</div>

<style>
	.container {
		height: 100vh;
		display: grid;
		place-items: center;
	}
	.sub-container {
		display: flex;
		flex-direction: column;
	}
	.options-container {
		display: flex;
		justify-content: center;
	}
	button,
	a {
		margin: 0 4px;
		border: 0;
		padding: 10px 18px;
		cursor: pointer;
		border-radius: 4px;
		font-size: 18px;
		color: #fff;
		transition: 0.25s;
		text-decoration: none;
	}
	button:hover,
	a:hover {
		opacity: 0.88;
	}
	.confirm {
		background-color: var(--error-color);
	}
	.cancel {
		background-color: var(--success-color);
	}
</style>
