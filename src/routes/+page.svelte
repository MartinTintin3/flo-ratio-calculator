<script>
	// Stuff
	let url = "";
	let data_promise = null;
	
	const get_data = async () => {
		// https://www.flowrestling.org/people/9079327-dillon-noonan
		const raw = await (await fetch(url)).text();
		const data = (new DOMParser()).parseFromString(raw, "text/html");

		alert(data.body.innerHTML);
	}
</script>

<input type="url">
<button type="button" on:click={() => data_promise = get_data(url)}>Fetch</button>

<p>
	{#if data_promise == null}
		{#await data_promise}
			Loading planet...
		{:then data}
			Your next planet is {data}.
		{:catch error}
			System error: {error.message}.
		{/await}
	{/if}
</p>

<style>
	* {
		text-align: center;
		margin: auto;
	}
</style>