<script>
	import { page } from '$app/stores';
	import { supabase } from '$lib/external/supabaseClient';
	import { onDestroy, onMount } from 'svelte';

	import Loader from '$lib/components/loader.svelte';
	import Notification from '$lib/components/notification.svelte';
	import { goto } from '$app/navigation';

	const id = $page.params.id;

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

	/**
	 * @type {import("@supabase/realtime-js").RealtimeSubscription | null}
	 */
	let profileSubscription;
	let addingToCart = false;
	let removingFromCart = false;

	let addedToCart = false;

	/**
	 * @type {import("@supabase/gotrue-js").User | null}
	 */
	let user;

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

	async function checkIfInCart() {
		const {
			data,
			error: _error,
			status
		} = await supabase.from('profiles').select('cart').eq('id', user?.id).single();

		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}

		addedToCart = data.cart.some((/** @type {{id: string, title: string}} */ product) => {
			return JSON.stringify({ id, title: productInfo?.title }) === JSON.stringify(product);
		});
	}

	onMount(async () => {
		supabase.auth.user();
		user = supabase.auth.user();

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
			user && checkIfInCart();
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

			if (user) {
				profileSubscription = supabase
					.from(`profiles:id=eq.${user?.id}`)
					.on('UPDATE', (payload) => {
						addedToCart = payload.new.cart
							? payload.new.cart.some((/** @type {{id: string, title: string}} */ product) => {
									return (
										JSON.stringify({ id, title: productInfo?.title }) === JSON.stringify(product)
									);
							  })
							: false;
					})
					.subscribe();
			}
		}
	});

	onDestroy(() => {
		productSubscription?.unsubscribe();
		profileSubscription?.unsubscribe();
	});

	async function addCart() {
		if (!user) {
			goto('/auth/signin');
			return;
		}

		if (addedToCart) {
			removingFromCart = true;

			const { error: _error } = await supabase.rpc('remove_array', {
				id: user.id,
				old_element: { id, title: productInfo?.title }
			});

			removingFromCart = false;
			if (_error) {
				error = `${_error.message}`;
				return;
			}
		} else {
			addingToCart = true;

			const { error: _error } = await supabase.rpc('append_array', {
				id: user.id,
				new_element: { id, title: productInfo?.title }
			});

			addingToCart = false;
			if (_error) {
				error = `${_error.message}`;
				return;
			}
		}
	}
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
				<img class="product-image" src={imageSrc} alt={productInfo?.title} />
			{/if}
		</div>

		<div class="product-info">
			<h1 class="product-title">{productInfo?.title}</h1>
			<p class="product-price">${productInfo?.price}</p>
			<p>Current Stock: {productInfo?.stock}</p>

			<p class="seller">
				Seller:
				{#if loadingUsername}
					<span class="loader-container"><Loader size={30} center={false} /></span>
				{:else}
					<b>{username}</b>
				{/if}
			</p>

			<button
				on:click={addCart}
				class={addedToCart ? 'remove-cart' : 'add-cart'}
				disabled={addingToCart || removingFromCart}
			>
				{#if addingToCart}
					Adding to Cart..
				{:else if removingFromCart}
					Removing from Cart..
				{:else if addedToCart}
					Remove from Cart
				{:else}
					Add to Cart
				{/if}
			</button>
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
	.product-image {
		max-width: 300px;
	}
	.seller {
		margin-bottom: 24px;
	}
	.add-cart {
		margin-top: 12px;
		background-color: #fff;
		border: 1px solid var(--secondary-color);
		border-radius: 4px;
		padding: 6px 12px;
		cursor: pointer;
		color: #000;
		transition: 0.25s;
	}
	.add-cart:hover {
		background-color: var(--secondary-color);
		color: #fff;
	}
	.add-cart:disabled,
	.add-cart[disabled] {
		cursor: default;
		background-color: var(--secondary-color);
		color: #fff;
		opacity: 0.5;
	}
	.remove-cart {
		margin-top: 12px;
		border: 0;
		background-color: var(--secondary-color);
		border-radius: 4px;
		padding: 6px 12px;
		cursor: pointer;
		color: #fff;
		transition: 0.25s;
	}
	.remove-cart:hover {
		opacity: 0.8;
	}
	.remove-cart:disabled,
	.remove-cart[disabled] {
		opacity: 0.5;
	}
</style>
