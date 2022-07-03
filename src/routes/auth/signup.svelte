<script>
	import { supabase } from '$lib/external/supabaseClient';
	import Notification from '$lib/components/notification.svelte';

	let submitted = false;
	let checkEmail = false;

	let email = '';
	let password = '';

	let error = '';

	let loading = false;

	async function signUp() {
		loading = true;
		submitted = true;

		const res = await supabase.auth.signUp({ email, password });
		if (res.error) {
			error = `[${res.error.status}] ${res.error.message}`;
			console.log(error);
		} else {
			checkEmail = true;
		}

		loading = false;
	}

	function inputChange() {
		error = '';
		submitted = false;
	}
</script>

<div class="container">
	<form class="form" on:submit|preventDefault={signUp}>
		<h1>Sign Up</h1>

		<input
			class="input"
			on:input={inputChange}
			disabled={checkEmail}
			type="text"
			required
			bind:value={email}
			placeholder="Email"
		/>
		<input
			class="input"
			on:input={inputChange}
			disabled={checkEmail}
			type="password"
			required
			bind:value={password}
			placeholder="Password"
		/>
		<input
			class="submit-btn"
			type="submit"
			value={loading ? 'Loading' : 'Sign Up'}
			disabled={!email || !password || submitted}
		/>

		<p class="login">
			Have an Account?
			<a href="/auth/signin">Sign In Instead</a>
		</p>
	</form>

	<Notification notification={error} show={!!error} type="error" />
	<Notification notification="Check Your Email" show={checkEmail} type="info" />
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
	.login {
		align-self: center;
		margin-top: 30px;
		font-size: 16px;
	}
</style>
