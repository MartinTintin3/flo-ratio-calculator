<script>
	// Stuff
	let url = "";
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
		// https://www.flowrestling.org/people/9079327-dillon-noonan
		const headers = new Headers({
			"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
			"Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8"
		});

		const raw = await (await fetch(url, { method: "GET", headers })).text();

		url = "";

		const parsedPage = (new DOMParser()).parseFromString(raw, "text/html");

		window["parsedPage"] = parsedPage;

		/**
		 * @type {Array<{season: String, total: number, wins: number, losses: number, pins: number, techs: number, ratio: [number, number]}>}
		 */
		let data = [];

		try {
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
	}
</script>

<div class="url-input">
	<input type="url" bind:value={url} placeholder="Flo Profile URL">
	<button type="button" on:click={() => get_data()}>Fetch</button>
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