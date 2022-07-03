<script>
	import { page } from '$app/stores';
	import Loader from '$lib/components/loader.svelte';
	import Notification from '$lib/components/notification.svelte';
	import { supabase } from '$lib/external/supabaseClient';

	import { onDestroy, onMount } from 'svelte';

	let error = '';
	let loading = true;
	let notFound = false;

	/**
	 * @type {import("@supabase/realtime-js").RealtimeSubscription | null}
	 */
	let productSubscription;

	/**
	 * @type {{title: string, price: number, stock: number, seller: string} | null}
	 */
	let productInfo;
	let updated = false;
	let deleted = false;

	let loadingUsername = false;
	/**
	 * @type {string | null}
	 */
	let username;

	const id = $page.params.id;

	onMount(async () => {
		const {
			data: productData,
			error: _error,
			status
		} = await supabase.from('products').select('title, price, stock, seller').eq('id', id).single();

		loading = false;
		if (_error) {
			if (_error.code === 'PGRST116' || _error.code === '22P02') {
				notFound = true;
			} else {
				error = `${status} ${_error.message}`;
			}

			return;
		} else {
			productInfo = productData;
			loadingUsername = true;

			const {
				data: userData,
				error: _error,
				status
			} = await supabase.from('profiles').select('username').eq('id', productData.seller).single();
			if (_error) {
				error = `${status} ${_error.message}`;
				return;
			}

			username = userData.username;
			loadingUsername = false;

			productSubscription = supabase
				.from(`products:id=eq.${id}`)
				.on('UPDATE', (payload) => {
					productInfo = payload.new;
					updated = true;

					setTimeout(() => (updated = false), 5000);
				})
				.on('DELETE', (_) => {
					productInfo = null;
					notFound = true;

					deleted = true;
				})
				.subscribe();
		}
	});

	onDestroy(() => productSubscription?.unsubscribe());
</script>

<svelte:head>
	<title>View Product</title>
</svelte:head>

{#if loading}
	<Loader />
{:else if notFound}
	<div class="center not-found">
		<h1>Product Not Found</h1>
		<a href="/">Go Home</a>
	</div>
{:else}
	<div class="product-container">
		<h1>{productInfo?.title}</h1>
		<p>${productInfo?.price}</p>
		{#if loadingUsername}
			<Loader size={20} center={false} />
		{:else}
			<p>{username}</p>
		{/if}
	</div>
{/if}

<Notification show={!!error} notification={error} type="error" />

<Notification show={updated} notification="This Product Updated" type="info" />
<Notification show={deleted} notification="This Product was Just Deleted" type="error" />

<style>
	.not-found {
		font-family: 'Raleway', sans-serif;
	}
	.product-container {
		padding: 20px 30px;
		font-family: 'Quicksand', sans-serif;
	}
</style>
