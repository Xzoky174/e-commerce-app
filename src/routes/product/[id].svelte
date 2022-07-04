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
	 * @type {{title: string, price: number, stock: number, seller: string, picture_url: string} | null}
	 */
	let productInfo;

	let updated = false;
	let deleted = false;

	let loadingUsername = false;
	/**
	 * @type {string | null}
	 */
	let username;

	let loadingImage = false;
	/**
	 * @type {string | null}
	 */
	let imageSrc;

	const id = $page.params.id;

	async function loadUsername() {
		if (!productInfo || !productInfo.seller) return;
		loadingUsername = true;

		const {
			data: userData,
			error: _error,
			status
		} = await supabase.from('profiles').select('username').eq('id', productInfo.seller).single();
		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}

		username = userData.username;
		loadingUsername = false;
	}

	async function loadImage() {
		if (!productInfo || !productInfo.picture_url) return;
		loadingImage = true;

		const { data, error: _error } = await supabase.storage
			.from('product-images')
			.download(productInfo.picture_url);

		loadingImage = false;
		if (_error) {
			error = `${_error.message}`;
			return;
		}
		if (!data) {
			error = 'Oops. Something Went Wrong!';
			return;
		}
		imageSrc = URL.createObjectURL(data);
	}

	onMount(async () => {
		const {
			data: productData,
			error: _error,
			status
		} = await supabase
			.from('products')
			.select('title, price, stock, seller, picture_url')
			.eq('id', id)
			.single();

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
			loadImage();
			loadUsername();

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
		<div class="product-img">
			{#if loadingImage}
				<Loader size={30} center={false} />
			{:else}
				<img src={imageSrc} alt={productInfo?.title} />
			{/if}
		</div>

		<div class="product-info">
			<h1 class="product-title">{productInfo?.title}</h1>
			<p class="product-price">${productInfo?.price}</p>

			<p>
				Seller:
				{#if loadingUsername}
					<span class="loader-container"><Loader size={30} center={false} /></span>
				{:else}
					<b>{username}</b>
				{/if}
			</p>
		</div>
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
		display: flex;
		gap: 30px;
	}
	.product-title {
		margin-bottom: 0;
	}
	.product-price {
		margin-top: 0;
	}
	.loader-container {
		display: inline-block;
	}
</style>
