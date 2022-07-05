<script>
	import { page } from '$app/stores';

	import { onMount, onDestroy } from 'svelte';

	import { supabase } from '$lib/external/supabaseClient';
	import Notification from '$lib/components/notification.svelte';
	import Loader from '$lib/components/loader.svelte';

	/**
	 * @type {import("@supabase/gotrue-js").User | null}
	 */
	let user;

	let username = '';
	let profileNotCreated = false;
	let isSeller = false;

	let loading = true;
	let error = '';

	/**
	 * @type {Array<{title: string, id: string, created_at: Date, price: number, stock: number, seller: string}>}
	 */
	let products = [];

	/**
	 * @type {import("@supabase/supabase-js").Subscription | null}
	 */
	let authListener;

	/**
	 * @type {import("@supabase/supabase-js").RealtimeSubscription}
	 */
	let productsListener;

	let signed_in = !!$page.url.searchParams.get('signed_in');
	let signed_out = !!$page.url.searchParams.get('signed_out');

	onMount(async () => {
		initUser();
		const { data: _authListener } = supabase.auth.onAuthStateChange(
			(_, session) => (user = session && session.user ? session.user : null)
		);
		authListener = _authListener;

		const { data, error: _error, status } = await supabase.from('products').select('*');
		loading = false;

		if (_error) {
			error = `${status} ${_error.message}`;
			return;
		}
		products = data;

		productsListener = supabase
			.from('products')
			.on('INSERT', (payload) => {
				products.push(payload.new);
				products = products;
			})
			.on(
				'UPDATE',
				(payload) =>
					(products = products.map((product) =>
						product.id === payload.old.id ? payload.new : product
					))
			)
			.on(
				'DELETE',
				(payload) => (products = products.filter((product) => product.id !== payload.old.id))
			)
			.subscribe();

		setTimeout(() => {
			signed_in = false;
			signed_out = false;
		}, 5000);
	});
	onDestroy(() => {
		authListener?.unsubscribe();
		productsListener && productsListener.unsubscribe();
	});

	async function initUser() {
		supabase.auth.user(); // First time may return null.
		user = supabase.auth.user();

		if (user && !profileNotCreated) {
			let {
				data,
				error: _error,
				status
			} = await supabase.from('profiles').select(`username, is_seller`).eq('id', user.id).single();

			if (_error) {
				if (_error.code === 'PGRST116') {
					// PGRST116: More than 1 or no items where returned when requesting a singular response.
					profileNotCreated = true;

					setTimeout(() => (profileNotCreated = false), 5000);
				} else {
					error = `${status} ${_error.message}`;
				}
			} else {
				username = data.username;
				isSeller = data.is_seller;
			}
		}
	}
</script>

<svelte:head>
	<title>Home</title>
</svelte:head>

<Notification show={signed_in} notification="Signed In Successfully" type="info" />
<Notification show={signed_out} notification="Signed Out Successfully" type="info" />
<Notification
	show={profileNotCreated}
	notification="You Still Haven't Completed Your Profile!"
	type="info"
/>
<Notification show={!!error} notification={error} type="error" />

{#if loading}
	<Loader />
{:else if products.length > 0}
	<h1>Products</h1>
	{#each products as product}
		<a href={`/product/${product.id}`}>{product.title} - ${product.price}</a>
	{/each}
{:else}
	<div class="center font-raleway">
		<h1>Uh Oh. We Couldn't Find Any Products!</h1>
		{#if isSeller}
			<p class="create-product">But you can <a href="/product/create">Create a Product</a>!</p>
		{/if}
	</div>
{/if}

<style>
	.create-product {
		font-size: 20px;
	}
</style>
