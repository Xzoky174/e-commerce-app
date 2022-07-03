<script>
	import { supabase } from '$lib/external/supabaseClient';
	import Notification from '$lib/components/notification.svelte';
	import { goto } from '$app/navigation';

	let submitted = false;

	let email = '';
	let password = '';

	let loading = false;

	let error = '';

	async function signIn() {
		loading = true;
		submitted = true;

		const res = await supabase.auth.signIn({ email, password });

		if (res.error) {
			error = `[${res.error.status}] ${res.error.message}`;
			console.log(error);
		} else {
			goto('/');
		}

		loading = false;
	}

	function inputChange() {
		error = '';
		submitted = false;
	}
</script>

<svelte:head>
	<title>Sign In</title>
</svelte:head>

<div class="container">
	<form class="form" on:submit|preventDefault={signIn}>
		<h1>Sign In</h1>

		<input
			class="input"
			on:input={inputChange}
			type="text"
			required
			bind:value={email}
			placeholder="Email"
		/>
		<input
			class="input"
			on:input={inputChange}
			type="password"
			required
			bind:value={password}
			placeholder="Password"
		/>
		<input
			class="submit-btn"
			type="submit"
			value={loading ? 'Loading' : 'Sign In'}
			disabled={!email || !password || submitted}
		/>

		<p class="signup">
			Have an Account?
			<a href="/auth/signup">Sign Up Instead</a>
		</p>
	</form>

	<Notification show={!!error} notification={error} type="error" />
</div>

<style>
	.container {
		width: 100%;
		height: 100vh;
		display: grid;
		place-items: center;
	}
	.form {
		display: flex;
		flex-direction: column;
		border: 1px solid var(--border-color);
		border-radius: 8px;
		padding: 60px 40px;
		padding-bottom: 30px;
	}
	.form > * {
		font-family: 'Quicksand', sans-serif;
	}
	.form h1 {
		margin: 0;
		margin-bottom: 28px;
	}
	.input {
		background: transparent;
		border: 2px solid var(--border-color);
		padding: 8px 10px;
		font-size: 20px;
		margin: 2px 0;
		border-radius: 4px;
	}
	.input:focus {
		outline: 0;
	}
	.submit-btn {
		align-self: center;
		margin-top: 24px;
		border: 0;
		color: #fff;
		background-color: var(--primary-color);
		padding: 8px 12px;
		border-radius: 4px;
		cursor: pointer;
		font-size: 18px;
		transition: 0.25s;
		font-weight: bold;
	}
	.submit-btn:hover {
		opacity: 0.8;
	}
	.submit-btn:disabled,
	.submit-btn[disabled] {
		cursor: default;
		opacity: 0.4;
	}
	.signup {
		align-self: center;
		margin-top: 30px;
		font-size: 16px;
	}
</style>
