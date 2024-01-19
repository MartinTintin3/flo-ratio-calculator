<script>
	import { page } from "$app/stores";
	import { onMount } from "svelte";

	let search_input = "";
	let url = "";

	let current_name = "";

	let loading = false;

	/**
	 * @type {Array<{season: String, total: number, wins: number, losses: number, pins: number, techs: number, ratio: [number, number]}>}
	 */
	let latest_data = [];

	/**
	 * @return {[number, number]}
	 * @param {number} numerator
	 * @param {number} denominator
	 */
	function reduce(numerator, denominator){
		/**
		 * @param {number} a
		 * @param {number} b
		 * @return {number}
		 */
		const get_gcd = (a,b) => {
			return b ? get_gcd(b, a%b) : a;
		};
		const gcd = get_gcd(numerator,denominator);
		return [numerator/gcd, denominator/gcd];
	}
	
	const get_data = async () => {
		loading = true;
		
		const headers = new Headers({
			"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
			"Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8"
		});

		if (!search_input.includes("flowrestling.org")) {
			const raw = await (await fetch(`https://www.flowrestling.org/search?q=${encodeURIComponent(search_input)}&page=1&type=person`, { method: "GET", headers })).text();

			const regex = /https:\/\/www\.flowrestling\.org\/people\/\d+-[^&]+/g;

			const options = Array.from(raw.matchAll(regex));

			if (options.length > 0) {
				url = options[0][0];
			} else {
				alert("No results found");
				loading = false;
				return;
			}
		} else {
			url = search_input;
		}

		// set query param "url" to the url
		history.pushState({}, null, `?url=${encodeURIComponent(url)}`);

		search_input = "";

		const raw = await (await fetch(url, { method: "GET", headers })).text();

		const parsedPage = (new DOMParser()).parseFromString(raw, "text/html");

		window["parsedPage"] = parsedPage;

		/**
		 * @type {Array<{season: String, total: number, wins: number, losses: number, pins: number, techs: number, ratio: [number, number]}>}
		 */
		let data = [];

		try {
			current_name = parsedPage.body.querySelector("body > flo-root > div > div.flex-1.ng-tns-c168-0.content-wrapper > ng-component > flo-collection-view > flo-header-layout > main > div > div.col-12.col-lg-8.pt-0.pt-sm-2.content-container > div.header-container > flo-collection > nav > h1").innerText;

			const list = Array.from(parsedPage.body.querySelector("body > flo-root > div > div.flex-1.ng-tns-c168-0.content-wrapper > ng-component > flo-collection-view > flo-header-layout > main > div > div.col-12.col-lg-8.pt-0.pt-sm-2.content-container > div.w-100 > ng-component > flo-collection-tab-view > flo-node-list").children).slice(1);
			console.log(list);
			for (const parsedItem of list) {
				/** 
				 * @type {{season: String, total: number, wins: number, losses: number, pins: number, techs: number, ratio: [number, number]}}
				 */
				const item = {
					season: parsedItem.children[0].children[0].children[0].children[0].innerText,
					total: 0,
					wins: 0,
					losses: 0,
					pins: 0,
					techs: 0,
					ratio: [0, 0],
				};

				const matches = Array.from(parsedItem.children[0].children[0].children[1].children[0].children[1].children[0].children[1].children)
				item.total = matches.length;
				for (const match of matches) {
					const type = match.children[1].children[0].children[0].children[0].innerText;
					if (type == "W") {
						item.wins++;

						const result = match.children[3].children[0].children[0].children[0].innerText.split(" ")[0];
						
						switch (result) {
							case "F":
								item.pins++;
								break;
							case "TF":
								item.techs++;
								break;
						}
					} else {
						item.losses++;
					}
				}

				item.ratio = item.wins == 0 ? [0, item.losses] : item.losses != 0 ? reduce(item.wins, item.losses) : [item.wins, item.losses];

				data.push(item);
			}
		} catch(e) {
			console.log(e);
			alert("Invalid url");
		}

		latest_data = data;

		loading = false;
	}

	onMount(() => {
		alert("Data fetched from flowrestling.org/people is not guaranteed to be accurate. A parser for flowrestling.org/athletes is in the works.")

		if ($page.url.searchParams.get("url") != "") {
			search_input = $page.url.searchParams.get("url");
			get_data();
		}
	});
</script>

<div class="url-input">
	<div>
		<input type="text" bind:value={search_input} placeholder="Flo URL or full name">
		<button type="button" on:click={() => get_data()}>Fetch</button>
	</div>
	<h2>{current_name}</h2>
</div>

{#each latest_data as item}
	<ul class="season-container">
		<h1>{item.season}</h1>
		<h2>{item.total} matches</h2>
		<h2 class="green">{item.wins} wins</h2>
		<h2 class="red">{item.losses} losses</h2>
		<h2>{item.pins} pins</h2>
		<h2>{item.techs} techs</h2>
		<h2>W/L Ratio <span class="green">{item.ratio[0]}</span>:<span class="red">{item.ratio[1]}</span> <span class="{item.ratio[0] > item.ratio[1] ? "green" : "red"}">({item.ratio[1] != 0 ? ((Math.round(item.ratio[0]/item.ratio[1] * 100) / 100).toFixed(2)) : item.ratio[0]})</span></h2>
	</ul>
{/each}

<style>
	* {
		text-align: center;
		margin: auto;
		font-family: Arial, Helvetica, sans-serif;
	}

	.url-input > * {
		padding: 0.1em;
	}

	.green {
		color: green;
	}

	.red {
		color: red;
	}

	.season-container {
		border: 1px solid black;
		padding: 10px;
		margin: 10px;
		display: inline-block;
	}
</style>