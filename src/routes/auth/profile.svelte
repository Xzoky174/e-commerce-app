<script>
	import { goto } from '$app/navigation';
	import { onMount } from 'svelte/internal';

	import { supabase } from '$lib/external/supabaseClient';
	import Notification from '$lib/components/notification.svelte';

	let loading = true;

	let email = '';
	let username = '';

	let error = '';
	let updated = false;

	/**
	 * @type {import("@supabase/gotrue-js").User | null}
	 */
	let user;

	async function updateProfile() {
		loading = true;

		const updates = {
			id: user?.id,
			username
		};

		const { error: _error, status } = await supabase
			.from('profiles')
			.upsert(updates, { returning: 'minimal' });
		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}

		updated = true;

		setTimeout(() => (updated = false), 5000);

		loading = false;
	}

	onMount(async () => {
		loading = true;

		supabase.auth.user();
		user = supabase.auth.user();

		if (!user) {
			goto('/auth/signin');
			return;
		}

		let {
			data,
			error: _error,
			status
		} = await supabase.from('profiles').select(`username`).eq('id', user.id).single();

		if (_error && _error.code !== 'PGRST116') {
			// PGRST116: More than 1 or no items where returned when requesting a singular response.
			error = `${status} ${_error.message}`;
		}
		if (data) username = data.username;

		email = user.email ?? '';

		loading = false;
	});
</script>

<svelte:head>
	<title>Edit Profile</title>
</svelte:head>

<form class="form" on:submit|preventDefault={updateProfile}>
	<div class="input-container">
		<label class="label" for="email">Email</label>
		<input class="input" id="email" type="text" value={email} disabled />
		<label class="label" for="username">Name</label>
		<input class="input" id="username" type="text" bind:value={username} />
	</div>

	<input
		type="submit"
		class="submit-btn"
		value={loading ? 'Loading..' : 'Update'}
		disabled={loading}
	/>
</form>

<Notification show={!!error} type="error" notification={error} />
<Notification show={updated} type="info" notification="Updated Successfully" />

<style>
	.form {
		padding: 20px 30px;
	}
	.form > * {
		font-family: 'Quicksand', sans-serif;
	}
	.label {
		font-weight: 600;
		font-size: 24px;
	}
	.input-container {
		display: grid;
		grid-template-columns: 80px 230px;
		align-items: center;
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
		margin-top: 18px;
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
</style>
