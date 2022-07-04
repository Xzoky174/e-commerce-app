<script>
	import { goto } from '$app/navigation';
	import { supabase } from '$lib/external/supabaseClient';
	import { onMount } from 'svelte';

	import Notification from '$lib/components/notification.svelte';
	import Loader from '$lib/components/loader.svelte';

	/**
	 * @type {import("@supabase/gotrue-js").User | null}
	 */
	let user;

	let error = '';
	let loading = true;
	let is_seller = false;

	let title = '';
	let price = 0;
	let stock = 1;

	/**
	 * @type {FileList | null | undefined}
	 */
	let files;
	let fileUploaded = true;

	let creating = false;

	onMount(async () => {
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
		} = await supabase.from('profiles').select(`is_seller`).eq('id', user.id).single();
		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}

		is_seller = data.is_seller;
		loading = false;
	});

	async function createProduct() {
		if (!files || !files[0]) {
			fileUploaded = false;
			return;
		}
		creating = true;

		const file = files[0];
		const filePath = `public/${title}.${file.name.split('.').pop()}`;

		const { error: storage_error } = await supabase.storage
			.from('product-images')
			.upload(filePath, file);
		if (storage_error) {
			creating = false;

			console.error(storage_error);
			error = `${storage_error.message}`;

			return;
		}

		const {
			data,
			error: _error,
			status
		} = await supabase
			.from('products')
			.insert({ title, price, stock, picture_url: filePath, seller: user?.id });

		creating = false;
		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}

		goto(`/product/${data[0].id}`);
	}
</script>

<svelte:head>
	<title>Create Product</title>
</svelte:head>

{#if loading}
	<Loader />
{:else if is_seller}
	<form class="form" on:submit|preventDefault={createProduct}>
		<div class="input-container">
			<label class="label" for="title">Title</label>
			<input required class="input" type="text" bind:value={title} id="title" name="title" />

			<label class="label" for="price">Price ($)</label>
			<input
				required
				class="input"
				type="number"
				step="0.01"
				bind:value={price}
				id="price"
				name="price"
			/>

			<label class="label" for="stock">Stock</label>
			<input required class="input" type="number" bind:value={stock} id="stock" name="stock" />

			<label class="label image-label" for="image"> Upload Picture </label>
			<span class="file-selected">
				{#if files && files[0]}
					<i>{files[0].name}</i>
				{:else if !fileUploaded}
					<p class="no-file-uploaded">Please Upload a Picture!</p>
				{/if}
			</span>
			<input class="image-input" bind:files type="file" id="image" name="image" />
		</div>

		<input
			type="submit"
			class="submit-btn"
			value={creating ? 'Creating..' : 'Create'}
			disabled={creating}
		/>
	</form>
{:else}
	<p class="not-registered center">
		You are not registered as a seller. Please
		<a href="mailto:alishariq.m2@gmail.com">Email the Owner</a>
		to register yourself as a seller.
	</p>
{/if}

<Notification show={!!error} notification={error} type="error" />

<style>
	.not-registered {
		font-family: 'Raleway', sans-serif;
		font-size: 20px;
	}
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
		grid-template-columns: 112px 230px;
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
	.image-label {
		margin-top: 12px;
		border: 1px solid var(--success-color);
		border-radius: 2px;
		font-size: 20px;
		display: inline-block;
		padding: 6px 12px;
		cursor: pointer;
		justify-self: start;
		transition: 0.25s;
	}
	.image-label,
	.file-selected {
		grid-column: 1/3;
	}
	.image-label:hover {
		background-color: var(--success-color);
		color: #fff;
	}
	.image-input {
		display: none;
	}
	.no-file-uploaded {
		margin: 0;
		margin-top: 2px;
		color: var(--error-color);
		font-weight: bold;
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
