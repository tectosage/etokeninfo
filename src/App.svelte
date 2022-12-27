<script>
	import {ChronikClient} from "chronik-client"
	import {encode, decode} from "ecashaddrjs"
	const chronik = new ChronikClient("https://chronik.be.cash/xec")
	let slpdb = 'https://tokendb.kingbch.com/q/'

	let etoken = false

	let tokenIdInput
	let viewedTokenId 

	let recordsPromise 
	let tokenRecords = {}
	let tokenInfo = {tokenId: ''}

	let featuredTokens = [
		'fb4233e8a568993976ed38a81c2671587c5ad09552dedefa78760deed6ff87aa',
		'7e7dacd72dcdb14e00a03dd3aff47f019ed51a6f1f4e4f532ae50692f62bc4e5',
		'54dc2ecd5251f8dfda4c4f15ce05272116b01326076240e2b9cc0104d33b1484',
		'f36e1b3d9a2aaf74f132fef3834e9743b945a667a4204e761b85f2e7b65fd41a',
		'28eb601e438b1df2f49b3d783f7b236496ad9c07e4af35e8d6c5050732ef030a',
		'58f4fa833b8559dae06339da7764105280a0a1def3b02d148ce041c8fa5a5afd'
	]

	let newTokensToday = 0
	let tokenVolume = 0 

	let currentSeconds = (Math.floor(Date.now() / 1000))
	let yesterdaySeconds = currentSeconds - (24*3600)

	let confirmedGenesisQuery = {
		"v": 3,
		"q": {
			"db": [
			"c"
			],
			"aggregate": [
			{
				"$match": {
				"slp.valid": true,
				"slp.detail.transactionType" : "GENESIS",
				"blk.t": {
					"$gte": yesterdaySeconds
				}
				}
			},
			{
				"$group": {
				"_id": "$blk.t",
				"count": {
					"$sum": 1
				}
				}
			}
			],
			"limit": 10000
		},
		"r": {
			"f": "[ .[] | {block_epoch: ._id, txs: .count} ]"
		}
	}

	let confirmedQuery = {
		"v": 3,
		"q": {
			"db": [
			"c"
			],
			"aggregate": [
			{
				"$match": {
				"slp.valid": true,
				"blk.t": {
					"$gte": yesterdaySeconds
				}
				}
			},
			{
				"$group": {
				"_id": "$blk.t",
				"count": {
					"$sum": 1
				}
				}
			}
			],
			"limit": 10000
		},
		"r": {
			"f": "[ .[] | {block_epoch: ._id, txs: .count} ]"
		}
	}


	let unconfirmedGenesisURL = 'https://tokendb.kingbch.com/q/ewogICJ2IjogMywKICAicSI6IHsKICAgICJkYiI6IFsKICAgICAgInUiCiAgICBdLAogICAgImFnZ3JlZ2F0ZSI6IFsKICAgICAgewogICAgICAgICIkbWF0Y2giOiB7CiAgICAgICAgICAic2xwLnZhbGlkIjogdHJ1ZSwKICAgICAgICAgICJzbHAuZGV0YWlsLnRyYW5zYWN0aW9uVHlwZSIgOiAiR0VORVNJUyIKICAgICAgICB9CiAgICAgIH0sCiAgICAgIHsKICAgICAgICAiJGdyb3VwIjogewogICAgICAgICAgIl9pZCI6ICIkYmxrLnQiLAogICAgICAgICAgImNvdW50IjogewogICAgICAgICAgICAiJHN1bSI6IDEKICAgICAgICAgIH0KICAgICAgICB9CiAgICAgIH0KICAgIF0sCiAgICAibGltaXQiOiAxMDAwMAogIH0sCiAgInIiOiB7CiAgICAiZiI6ICJbIC5bXSB8IHtibG9ja19lcG9jaDogLl9pZCwgdHhzOiAuY291bnR9IF0iCiAgfQp9'
	let unconfirmedURL = 'https://tokendb.kingbch.com/q/ewogICJ2IjogMywKICAicSI6IHsKICAgICJkYiI6IFsKICAgICAgInUiCiAgICBdLAogICAgImFnZ3JlZ2F0ZSI6IFsKICAgICAgewogICAgICAgICIkbWF0Y2giOiB7CiAgICAgICAgICAic2xwLnZhbGlkIjogdHJ1ZQogICAgICAgIH0KICAgICAgfSwKICAgICAgewogICAgICAgICIkZ3JvdXAiOiB7CiAgICAgICAgICAiX2lkIjogIiRibGsudCIsCiAgICAgICAgICAiY291bnQiOiB7CiAgICAgICAgICAgICIkc3VtIjogMQogICAgICAgICAgfQogICAgICAgIH0KICAgICAgfQogICAgXSwKICAgICJsaW1pdCI6IDEwMDAwCiAgfSwKICAiciI6IHsKICAgICJmIjogIlsgLltdIHwge2Jsb2NrX2Vwb2NoOiAuX2lkLCB0eHM6IC5jb3VudH0gXSIKICB9Cn0='

	async function genesis(){
		let confirmedResult =  fetch(slpdb + btoa(JSON.stringify(confirmedGenesisQuery)))
		let unconfirmedResult =  fetch(unconfirmedGenesisURL)
		let confirmedJSON = await(await confirmedResult).json()
		// console.log(confirmedJSON)
		for(let i=0; i<confirmedJSON.c.length; i++){
			newTokensToday += confirmedJSON.c[i].txs
		}
		let unconfirmedJSON = await(await unconfirmedResult).json()
		for(let i=0; i<unconfirmedJSON.u.length; i++){
			newTokensToday += unconfirmedJSON.u[i].txs
		}
		// console.log(unconfirmedJSON)
	}
	async function volume(){
		let confirmedResult =  fetch(slpdb + btoa(JSON.stringify(confirmedQuery)))
		let unconfirmedResult =  fetch(unconfirmedURL)
		let confirmedJSON = await(await confirmedResult).json()
		// console.log(confirmedJSON)
		for(let i=0; i<confirmedJSON.c.length; i++){
			tokenVolume += confirmedJSON.c[i].txs
		}
		let unconfirmedJSON = await(await unconfirmedResult).json()
		for(let i=0; i<unconfirmedJSON.u.length; i++){
			tokenVolume += unconfirmedJSON.u[i].txs
		}
		// console.log(unconfirmedJSON)
	}
	async function makeRecords(){
		let promises = []
		for(let i=0; i<featuredTokens.length;i++){
			promises.push(chronik.token(featuredTokens[i]))
		}
		for(let i=0; i<promises.length;i++){
			let result = await promises[i]
			console.log(result)
			let slpData = result.slpTxData.genesisInfo
			tokenRecords[result.slpTxData.slpMeta.tokenId] = {ticker: slpData.tokenTicker,
				name:slpData.tokenName, uri: slpData.tokenDocumentUrl, hash: slpData.tokenDocumentHash,
				decimals: slpData.decimals, minted: result.tokenStats.totalMinted, burned: result.tokenStats.totalBurned, circulating: result.tokenStats.totalMinted - result.tokenStats.totalBurned, date: result.block.timestamp
			}
		}
	}

	async function viewToken(tokenId){
		try{
			if(!tokenRecords[tokenId]){
				console.log('creating new token record')
				let result = await chronik.token(tokenId)
				let slpData = result.slpTxData.genesisInfo
				tokenRecords[tokenId] = {ticker: slpData.tokenTicker,
				name:slpData.tokenName, uri: slpData.tokenDocumentUrl, hash: slpData.tokenDocumentHash,
				decimals: slpData.decimals, minted: result.tokenStats.totalMinted, burned: result.tokenStats.totalBurned, circulating: result.tokenStats.totalMinted - result.tokenStats.totalBurned, date: result.block.timestamp
			}
			}else{console.log('found token record')}
			viewedTokenId = tokenId
			if(tokenInfo.tokenId != viewedTokenId){
				tokenInfo.tokenId = tokenId
				tokenInfo.promise = getTokenInfo()
			}
		}catch (error){console.log(error)}
	}

	async function getTokenInfo(){
		console.log('getting token info')
		let holdersQuery= {
			"v": 3,
			"q": {
				"db": ["g"],
				"aggregate": [ 
					{ "$match": {
					"graphTxn.outputs": 
						{ "$elemMatch": {
						"status": "UNSPENT", 
						"slpAmount": { "$gte": 0 }
						}
					},
					"tokenDetails.tokenIdHex": viewedTokenId
					}
				},
					{ "$unwind": "$graphTxn.outputs" },
					{ "$match": {
						"graphTxn.outputs.status": "UNSPENT", 
						"graphTxn.outputs.slpAmount": { "$gte": 0 },
						"tokenDetails.tokenIdHex": viewedTokenId
					}
					},
					{ "$project": 
					{ "token_balance": "$graphTxn.outputs.slpAmount",
					"address": "$graphTxn.outputs.address",
					"txid": "$graphTxn.txid", 
					"vout": "$graphTxn.outputs.vout", 
					"tokenId": "$tokenDetails.tokenIdHex" }},
				{ "$group": { "_id": "$address", "token_balance": { "$sum": "$token_balance" }}}
				],
				"sort": {"token_balance" : -1},
				"limit" : 999999
				
			}
		}
		let recentTransactionsQuery = {
			"v": 3,
			"q": {
				"db": ["c", "u"],
				"find":
				{
				"$query":
				{
					"slp.detail.tokenIdHex": viewedTokenId
				},
				"$orderby":
				{
					"blk.i": -1
				}
				},
				"limit": 10
			},
			"r": {
				"f": "[.[] | { txid: .tx.h, tokenDetails: .slp } ]"
			}
		}
		let pastDayQuery = {
			"v": 3,
			"q": {
				"db": [
				"c"
				],
				"aggregate": [
				{
					"$match": {
					"slp.valid": true,
					"slp.detail.tokenIdHex" : viewedTokenId,
					"blk.t": {
						"$gte": yesterdaySeconds
					}
					}
				},
				{
					"$group": {
					"_id": "$blk.t",
					"count": {
						"$sum": 1
					}
					}
				}
				],
				"limit": 10000
			},
			"r": {
				"f": "[ .[] | {block_epoch: ._id, txs: .count} ]"
			}
		}
		let pastDayUnconfirmedQuery = {
			"v": 3,
			"q": {
				"db": [
				"u"
				],
				"aggregate": [
				{
					"$match": {
					"slp.valid": true,
					"slp.detail.tokenIdHex" : viewedTokenId
					}
				},
				{
					"$group": {
					"_id": "$blk.t",
					"count": {
						"$sum": 1
					}
					}
				}
				],
				"limit": 10000
			},
			"r": {
				"f": "[ .[] | {block_epoch: ._id, txs: .count} ]"
			}
		}
		let totalTransactionsQuery = {
			"v": 3,
			"q": {
				"db": ["g"],
				"aggregate": [
				{
					"$match": {
					"tokenDetails.tokenIdHex": viewedTokenId
					}
				},
				{
					"$group": {
					"_id": null,
					"count": {
						"$sum": 1
					}
					}
				}
				],
				"limit": 1
			},
			"r": {
				"f": "[ .[] | {count: .count} ]"
			}
		}
		let holders = fetch(slpdb + btoa(JSON.stringify(holdersQuery)))
		let recentTransactions = fetch(slpdb + btoa(JSON.stringify(recentTransactionsQuery)))
		let total = fetch(slpdb + btoa(JSON.stringify(totalTransactionsQuery)))
		let pastDayConfirmed = fetch(slpdb + btoa(JSON.stringify(pastDayQuery)))
		let pastDayUnconfirmed = fetch(slpdb + btoa(JSON.stringify(pastDayUnconfirmedQuery)))


		holders = await (await holders).json()
		recentTransactions = await (await recentTransactions).json()
		total = await (await total).json()
		pastDayConfirmed = await (await pastDayConfirmed).json()
		pastDayUnconfirmed = await (await pastDayUnconfirmed).json()

		tokenInfo.recent = [...recentTransactions.u, ...recentTransactions.c].slice(0,10)
		for(let i=0;i<tokenInfo.recent.length;i++){
			tokenInfo.recent[i].transaction = chronik.tx(tokenInfo.recent[i].txid)
		}
		for(let i=0;i<tokenInfo.recent.length;i++){
			tokenInfo.recent[i].transaction = await tokenInfo.recent[i].transaction
		}
		console.log(tokenInfo.recent)
		tokenInfo.topHolders = holders.g.slice(0,10)
		tokenInfo.holders = holders.g.length
		tokenInfo.total = total.g[0].count
		let count = 0
		pastDayConfirmed.c.map(x=> count += x.txs)
		pastDayUnconfirmed.u.map(x=> count += x.txs)
		tokenInfo.pastDay = count
		console.log(tokenRecords[viewedTokenId])
		console.log(tokenInfo)
	}

	recordsPromise = makeRecords()
	genesis()
	volume()
</script>

<main>
	<div id="iconsbar">
		<span id="logo"><button on:click={()=> viewedTokenId=null}><img alt="logo" src='/favicon.png'></button>
			
			
		</span>
		{#if !viewedTokenId}
		<span id="github"><svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
			<path d="M12 0C5.373 0 0 5.373 0 12C0 17.623 3.872 22.328 9.092 23.63C9.036 23.468 9 23.28 9 23.047V20.996C8.513 20.996 7.697 20.996 7.492 20.996C6.671 20.996 5.941 20.643 5.587 19.987C5.194 19.258 5.126 18.143 4.152 17.461C3.863 17.234 4.083 16.975 4.416 17.01C5.031 17.184 5.541 17.606 6.021 18.232C6.499 18.859 6.724 19.001 7.617 19.001C8.05 19.001 8.698 18.976 9.308 18.88C9.636 18.047 10.203 17.28 10.896 16.918C6.9 16.507 4.993 14.519 4.993 11.82C4.993 10.658 5.488 9.534 6.329 8.587C6.053 7.647 5.706 5.73 6.435 5C8.233 5 9.32 6.166 9.581 6.481C10.477 6.174 11.461 6 12.495 6C13.531 6 14.519 6.174 15.417 6.483C15.675 6.17 16.763 5 18.565 5C19.297 5.731 18.946 7.656 18.667 8.594C19.503 9.539 19.995 10.66 19.995 11.82C19.995 14.517 18.091 16.504 14.101 16.917C15.199 17.49 16 19.1 16 20.313V23.047C16 23.151 15.977 23.226 15.965 23.315C20.641 21.676 24 17.236 24 12C24 5.373 18.627 0 12 0Z" fill="black"/>
			</svg><a href="https://github.com/tectosage/etokeninfo" target="_blank" rel="noreferrer">tectosage/etokeninfo</a>
		</span>
		{:else}
		<button id="back" on:click={()=> viewedTokenId = null}><svg width="36" height="36" viewBox="0 0 36 36" fill="none" xmlns="http://www.w3.org/2000/svg">
			<path fill-rule="evenodd" clip-rule="evenodd" d="M3 18C3 17.1716 3.67158 16.5 4.5 16.5H31.5C32.3284 16.5 33 17.1716 33 18C33 18.8284 32.3284 19.5 31.5 19.5H4.5C3.67158 19.5 3 18.8284 3 18Z" fill="#2F2F2F"/>
			<path fill-rule="evenodd" clip-rule="evenodd" d="M14.5607 7.93933C15.1464 8.52513 15.1464 9.47487 14.5607 10.0607L6.62131 18L14.5607 25.9393C15.1464 26.5251 15.1464 27.4749 14.5607 28.0606C13.9749 28.6464 13.0251 28.6464 12.4393 28.0606L3.43933 19.0606C2.85356 18.4749 2.85356 17.5251 3.43933 16.9393L12.4393 7.93933C13.0251 7.35356 13.9749 7.35356 14.5607 7.93933Z" fill="#2F2F2F"/>
			</svg>
			</button>
		{/if}
	</div>


{#if !viewedTokenId}

	<div id="container">
		<span id="input"><input id="tokenid" placeholder="etoken id" spellcheck="false" bind:value={tokenIdInput}><button id="search" on:click={()=> viewToken(tokenIdInput)}>→</button></span>
		<span id="recommended">Recommended eTokens</span>
	<div id="featuredTokens">
		{#await recordsPromise}
		<span id="homeLoading">Loading</span>
		{:then}
		{#each featuredTokens as token}
		<button class="featuredToken" on:click={()=>viewToken(token)}>
			<div class="icon"><span><img alt="token icon" src={"https://etoken-icons.s3.us-west-2.amazonaws.com/64/"+token+'.png'}  onerror="this.onerror=null; this.remove();"><span class="tick">{tokenRecords[token].ticker.slice(0,1).toUpperCase()}</span></span></div>

			<span class="name">{tokenRecords[token].name}</span>
			<span class="tokenid">TokenId: {token.slice(0,5)}...{token.slice(-5)}</span>
		</button>
		{/each}
		{/await}
	</div>

	<span id="data">Data Overview</span>

	<span id="eTokenVolume"><svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
		<path d="M2 2V21C2 21.55 2.45 22 3 22H22" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M18 5C18 6.15 17.9 7.19 17.71 8.14" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M8 17C10.28 16.88 14.12 15.62 16.32 11.88" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M6 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M12 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M18 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M2 5H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M2 11H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M2 17H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
		<path d="M19 14C20.1046 14 21 13.1046 21 12C21 10.8954 20.1046 10 19 10C17.8954 10 17 10.8954 17 12C17 13.1046 17.8954 14 19 14Z" fill="#AEB5BC"/>
		<path d="M17 12C18.1046 12 19 11.1046 19 10C19 8.89543 18.1046 8 17 8C15.8954 8 15 8.89543 15 10C15 11.1046 15.8954 12 17 12Z" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round"/>
		</svg>
		 &nbsp;eToken transactions today:&nbsp; <span class="black">{tokenVolume}</span></span>
	<span id="newTokens">
		<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
			<g clip-path="url(#clip0_8_243)">
			<path fill-rule="evenodd" clip-rule="evenodd" d="M4 3C4.55228 3 5 3.44772 5 4V23H24C24.5523 23 25 23.4477 25 24C25 24.5523 24.5523 25 24 25H4C3.44772 25 3 24.5523 3 24V4C3 3.44772 3.44772 3 4 3Z" fill="#F5F5F5"/>
			<path fill-rule="evenodd" clip-rule="evenodd" d="M2 1C2.55228 1 3 1.44772 3 2V21H22C22.5523 21 23 21.4477 23 22C23 22.5523 22.5523 23 22 23H2C1.44772 23 1 22.5523 1 22V2C1 1.44772 1.44772 1 2 1Z" fill="#AEB5BC"/>
			<path d="M8 9C7.44772 9 7 9.44772 7 10V20C7 20.5523 7.44772 21 8 21H10C10.5523 21 11 20.5523 11 20V10C11 9.44772 10.5523 9 10 9H8Z" fill="#F5F5F5"/>
			<path d="M6 7C5.44772 7 5 7.44772 5 8V18C5 18.5523 5.44772 19 6 19H8C8.55228 19 9 18.5523 9 18V8C9 7.44772 8.55228 7 8 7H6Z" fill="#AEB5BC"/>
			<path d="M14 3C13.4477 3 13 3.44772 13 4V20C13 20.5523 13.4477 21 14 21H16C16.5523 21 17 20.5523 17 20V4C17 3.44772 16.5523 3 16 3H14Z" fill="#F5F5F5"/>
			<path d="M12 1C11.4477 1 11 1.44772 11 2V18C11 18.5523 11.4477 19 12 19H14C14.5523 19 15 18.5523 15 18V2C15 1.44772 14.5523 1 14 1H12Z" fill="#AEB5BC"/>
			<path d="M20 13C19.4477 13 19 13.4477 19 14V20C19 20.5523 19.4477 21 20 21H22C22.5523 21 23 20.5523 23 20V14C23 13.4477 22.5523 13 22 13H20Z" fill="#F5F5F5"/>
			<path d="M18 11C17.4477 11 17 11.4477 17 12V18C17 18.5523 17.4477 19 18 19H20C20.5523 19 21 18.5523 21 18V12C21 11.4477 20.5523 11 20 11H18Z" fill="#AEB5BC"/>
			</g>
			<defs>
			<clipPath id="clip0_8_243">
			<rect width="24" height="24" fill="white"/>
			</clipPath>
			</defs>
		</svg>
		 <span>&nbsp;new eTokens today:&nbsp; <span class="black">{newTokensToday}</span></span>

	</div>



	<span id="credit">&nbsp;Build by&nbsp;<a id="tecto" href="https://twitter.com/thetectosage" target="_blank" rel="noreferrer">@thetectosage</a> &nbsp;Design by&nbsp;<a href="https://twitter.com/alitayin" target="_blank" rel="noreferrer">@alitayin</a> </span>
{:else}

	{#await tokenInfo.promise}
	<span id='infoLoading'>Loading token information...</span>
	{:then}
	<div id="infoContainer">
		<div id="topInfo">
			<div id="recentContainer">
				<span class="black">Recent Transactions</span>
				<div id="recentTransactions">
				{#each tokenInfo.recent as tx}
				<a href={"https://explorer.e.cash/tx/" + tx.txid} class="recentTransaction" target="_blank" rel="noreferrer">
					<div>
					<span class="transactionTop">
						<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
							<path fill-rule="evenodd" clip-rule="evenodd" d="M14.4629 8.12289L12.34 6H3C2.45 6 2 6.45 2 7V21C2 21.55 2.45 22 3 22H17C17.55 22 18 21.55 18 21V11.66L15.8771 9.53711L9.70711 15.7071C9.31658 16.0976 8.68342 16.0976 8.29289 15.7071C7.90237 15.3166 7.90237 14.6834 8.29289 14.2929L14.4629 8.12289Z" fill="#AEB5BC"/>
							<path fill-rule="evenodd" clip-rule="evenodd" d="M15 2C14.4477 2 14 2.44772 14 3C14 3.55228 14.4477 4 15 4H18.5858L14.4629 8.12289L15.8771 9.53711L20 5.41421V9C20 9.55228 20.4477 10 21 10C21.5523 10 22 9.55228 22 9V3C22 2.72942 21.8925 2.48394 21.718 2.30391C21.7108 2.29651 21.7035 2.28922 21.6961 2.28205C21.6036 2.19229 21.498 2.12396 21.3856 2.07705C21.2669 2.02742 21.1367 2 21 2H15Z" fill="#AEB5BC"/>
						</svg>
						<span class="black">tx id: {tx.txid.slice(0,7)}...{tx.txid.slice(-7)}</span>
						<span class="black">{(tx.transaction.outputs[1].slpToken.amount / (10 **tokenRecords[viewedTokenId].decimals)).toLocaleString()} {tokenRecords[viewedTokenId].ticker}</span>
					</span>
					<span class="transactionBottom">
						<span>{new Date(tx.transaction.timeFirstSeen*1000).toLocaleString()}</span>
						<span>
							{#if tx.transaction.block}
							Block: {tx.transaction.block.height}
							{:else}
							unconfirmed
							{/if}
						</span>
					</span>
					</div>
				</a>
				{/each}
				</div>
			</div>
			<div class="black" id="basicInfo">
				<span id="topTokenName">{tokenRecords[viewedTokenId].name} Basic Info</span>
				<div>
					<span id="tokenName">	<div class="icon small"><span><img alt="token icon" src={"https://etoken-icons.s3.us-west-2.amazonaws.com/64/"+viewedTokenId+'.png'}  onerror="this.onerror=null; this.remove();"><span class="tick">{tokenRecords[viewedTokenId].ticker.slice(0,1).toUpperCase()}</span></span></div> <span>{tokenRecords[viewedTokenId].name}</span>
					</span>
					<span>
						<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
							<path fill-rule="evenodd" clip-rule="evenodd" d="M14.6353 2.43058C16.5553 0.52314 19.6449 0.52314 21.5649 2.43058L21.5672 2.43289C22.5233 3.38904 23.0001 4.64722 23.0001 5.9C23.0001 7.15278 22.5233 8.41096 21.5672 9.36711L18.4872 12.4471C18.0967 12.8376 17.4635 12.8376 17.073 12.4471C16.6824 12.0566 16.6824 11.4234 17.073 11.0329L20.153 7.95289C20.7168 7.38904 21.0001 6.64722 21.0001 5.9C21.0001 5.15329 20.7172 4.41197 20.1541 3.84827C19.0145 2.71737 17.1861 2.71724 16.0464 3.84788L10.9526 8.95173C10.3826 9.51428 10.1001 10.2536 10.1001 11C10.1001 11.1344 10.1103 11.2699 10.129 11.4169C10.1401 11.4623 10.1457 11.4981 10.1488 11.5198C10.1505 11.5316 10.1519 11.5428 10.1531 11.5536C10.1825 11.6899 10.2163 11.81 10.2584 11.9243C10.262 11.9341 10.2655 11.9439 10.2688 11.9538C10.3993 12.3455 10.6248 12.718 10.949 13.0348L10.9572 13.0428C11.1246 13.2103 11.3007 13.3482 11.4746 13.4525C11.9482 13.7367 12.1017 14.3509 11.8176 14.8245C11.5334 15.2981 10.9192 15.4516 10.4456 15.1675C10.121 14.9728 9.81859 14.7321 9.54704 14.4612C8.99733 13.9227 8.60589 13.282 8.37634 12.6009C8.28744 12.3569 8.22627 12.1199 8.17951 11.8861C8.17785 11.8778 8.17629 11.8694 8.17484 11.8611C8.16888 11.8348 8.16398 11.8082 8.16015 11.7814C8.12574 11.5406 8.1001 11.2811 8.1001 11C8.1001 9.74748 8.57678 8.48785 9.54518 7.5307L14.6353 2.43058ZM8.1601 11.69C8.1601 11.6922 8.1601 11.6944 8.16012 11.6967L8.1601 11.69Z" fill="#2F2F2F"/>
							<path fill-rule="evenodd" clip-rule="evenodd" d="M12.1825 9.17551C12.4667 8.70192 13.0809 8.54836 13.5545 8.83251C13.8791 9.02724 14.1815 9.26791 14.4531 9.53885C15.0028 10.0774 15.3942 10.718 15.6238 11.3991C15.7127 11.6431 15.7738 11.8801 15.8206 12.1139C15.8222 12.1222 15.8238 12.1306 15.8253 12.1389C15.8312 12.1652 15.8361 12.1918 15.8399 12.2186C15.8744 12.4594 15.9 12.7189 15.9 13C15.9 14.2525 15.4233 15.5122 14.4549 16.4693L9.36479 21.5694C7.44481 23.4769 4.3552 23.4769 2.43522 21.5694L2.43289 21.5671C1.47674 20.611 1 19.3528 1 18.1C1 16.8472 1.47674 15.589 2.43289 14.6329L5.51289 11.5529C5.90342 11.1624 6.53658 11.1624 6.92711 11.5529C7.31763 11.9434 7.31763 12.5766 6.92711 12.9671L3.84711 16.0471C3.28326 16.611 3 17.3528 3 18.1C3 18.8467 3.28287 19.588 3.84595 20.1517C4.98554 21.2826 6.81395 21.2828 7.95368 20.1521C7.95419 20.1516 7.9547 20.1511 7.95522 20.1506L11.5319 16.5639L13.0475 15.0483C13.6175 14.4857 13.9 13.7464 13.9 13C13.9 12.8656 13.8897 12.7301 13.8711 12.5831C13.8599 12.5377 13.8544 12.5019 13.8513 12.4802C13.8496 12.4684 13.8482 12.4572 13.847 12.4464C13.8176 12.3101 13.7838 12.19 13.7417 12.0757C13.7381 12.0659 13.7346 12.0561 13.7313 12.0462C13.6007 11.6545 13.3752 11.282 13.0511 10.9652L13.0428 10.9572C12.8754 10.7897 12.6993 10.6518 12.5255 10.5475C12.0519 10.2633 11.8984 9.64909 12.1825 9.17551ZM15.84 12.31C15.84 12.3078 15.84 12.3056 15.84 12.3033V12.31Z" fill="#2F2F2F"/>
						</svg>
						<span>
							{#if tokenRecords[viewedTokenId].uri.slice(0,4) == "http"}
							<a href={tokenRecords[viewedTokenId].uri} target="_blank" rel="noreferrer">{tokenRecords[viewedTokenId].uri} </a>
							{:else}
							<a href={"http://" + tokenRecords[viewedTokenId].uri} target="_blank" rel="noreferrer">{tokenRecords[viewedTokenId].uri}</a>
							{/if}
						</span>
					</span>
					<span>
						<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
								<path fill-rule="evenodd" clip-rule="evenodd" d="M14.2246 3.88788L15.7177 4.91765L13.9604 5.50342L13.3747 7.2607L12.3449 5.76754L10.457 5.81998L11.5989 4.29751L11.0462 2.58914L12.7545 3.14185L14.277 2L14.2246 3.88788Z" fill="#2F2F2F"/>
								<path fill-rule="evenodd" clip-rule="evenodd" d="M5.76754 12.345L7.2607 13.3748L5.50342 13.9606L4.91765 15.7179L3.88788 14.2247L2 14.2771L3.14185 12.7547L2.58914 11.0463L4.29751 11.599L5.81998 10.4572L5.76754 12.345Z" fill="#2F2F2F"/>
								<path fill-rule="evenodd" clip-rule="evenodd" d="M6.96852 2.47134L6.93709 4.70268L8.70738 5.97605L6.62773 6.62788L5.9759 8.70754L4.70253 6.93724L2.47119 6.96867L3.81147 5.19202L3.08989 3.09005L5.19187 3.81162L6.96852 2.47134Z" fill="#2F2F2F"/>
								<path d="M21.49 21.49C20.81 22.17 19.7 22.17 19.02 21.49L9.24999 11.72C8.56999 11.04 8.56999 9.92999 9.24999 9.24999C9.92999 8.56999 11.04 8.56999 11.72 9.24999L21.49 19.02C22.17 19.7 22.17 20.8 21.49 21.49Z" fill="#2F2F2F"/>
								<path d="M12.605 15.0749L15.075 12.6049" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
								<path d="M9.2668 9.2334C9.26117 9.23889 9.25556 9.24442 9.24999 9.24999C8.56999 9.92999 8.56999 11.04 9.24999 11.72L12.6049 15.0749L15.0749 12.6049L11.72 9.24999C11.0726 8.60259 10.0354 8.57155 9.35066 9.15688C9.31838 9.18516 9.29017 9.21088 9.2668 9.2334Z" fill="#2F2F2F"/>
						</svg>
						<span>{(tokenRecords[viewedTokenId].minted / (10**tokenRecords[viewedTokenId].decimals)).toLocaleString()}</span>
					</span>
					<span class="push">
						<svg width="18" height="21" viewBox="0 0 18 21" fill="none" xmlns="http://www.w3.org/2000/svg">
							<path d="M8.6825 0C8.9625 1.11 8.9325 7.8 5.77254 8.56C5.77254 8.56 3.95254 8.36 2.74254 4.62C0.512541 7.31 -0.767459 11.39 0.502541 14.47C2.07254 18.29 5.79254 19.44 6.40254 19.62C9.2925 20.46 12.8625 19.97 15.3325 17.58C18.1525 14.86 18.0225 11.14 17.9825 10.38C17.7225 5.85 14.0125 1.72 8.6825 0Z" fill="#2F2F2F"/>
						</svg>
						<span>{(tokenRecords[viewedTokenId].burned / (10**tokenRecords[viewedTokenId].decimals)).toLocaleString()}</span>
					</span>
					<span class="push">
						<svg width="18" height="23" viewBox="0 0 18 23" fill="none" xmlns="http://www.w3.org/2000/svg">
							<g clip-path="url(#clip0_27_7)">
							<path d="M10.5901 3C10.5901 3 15.6601 10.6 17.1001 14.56C17.2401 14.94 17.4801 15.66 17.4701 16.61C17.4601 18.6 16.3401 20.65 14.6301 21.84C12.1601 23.55 8.6001 23.36 6.31011 21.4C4.82011 20.12 4.03011 18.2 4.00011 16.47C3.98011 15.53 4.19011 14.8 4.34011 14.35C5.61011 10.42 10.5901 3 10.5901 3Z" fill="#EAEAEA"/>
							<path d="M8.5901 1C8.5901 1 13.6601 8.6 15.1001 12.56C15.2401 12.94 15.4801 13.66 15.4701 14.61C15.4601 16.6 14.3401 18.65 12.6301 19.84C10.1601 21.55 6.60011 21.36 4.31011 19.4C2.82011 18.12 2.03011 16.2 2.00011 14.47C1.98011 13.53 2.19011 12.8 2.34011 12.35C3.61011 8.42 8.5901 1 8.5901 1Z" stroke="#2F2F2F" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
							<path d="M5.05 13.3C5.02 13.48 5 13.7 5 13.93C5.01 14.9 5.45 15.96 6.28 16.67C6.91 17.22 7.71 17.51 8.53 17.56" stroke="#2F2F2F" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
							</g>
							<defs>
							<clipPath id="clip0_27_7">
							<rect width="18" height="23" fill="white"/>
							</clipPath>
							</defs>
						</svg>
							
						<span>{(tokenRecords[viewedTokenId].circulating / (10**tokenRecords[viewedTokenId].decimals)).toLocaleString()}</span>
					</span>
					<span class="push">
						<svg width="20" height="21" viewBox="0 0 20 21" fill="none" xmlns="http://www.w3.org/2000/svg">
							<g clip-path="url(#clip0_27_25)">
							<path d="M11 18H2C0.9 18 0 17.1 0 16V7H18V10" fill="#2F2F2F"/>
							<path d="M16 2H2C0.9 2 0 2.9 0 4V7H18V4C18 2.9 17.1 2 16 2Z" fill="#2F2F2F"/>
							<path d="M0 7H18" stroke="white" stroke-width="2" stroke-miterlimit="10"/>
							<path d="M13 1V4" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linecap="round" stroke-linejoin="round"/>
							<path d="M5 1V4" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linecap="round" stroke-linejoin="round"/>
							<path d="M2 11H4" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M6 11H8" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M10 11H12" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M14 11H16" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M10 15H12" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M2 15H4" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path d="M6 15H8" stroke="white" stroke-width="2" stroke-miterlimit="10" stroke-linejoin="round"/>
							<path fill-rule="evenodd" clip-rule="evenodd" d="M16.4761 12.2316L19.0651 13.7369L15.9684 19.0937L13.1941 20.9331L13.3745 17.5969L16.4761 12.2316Z" fill="#2F2F2F"/>
							</g>
							<defs>
							<clipPath id="clip0_27_25">
							<rect width="20" height="21" fill="white"/>
							</clipPath>
							</defs>
							</svg>
							
						<span>{new Date(tokenRecords[viewedTokenId].date * 1000).toLocaleDateString(undefined, {month: 'long', day: 'numeric', year: "numeric"})}</span>
					</span>
				</div>
			</div>
		</div>
		<div id="bottomInfo">
			<div id="tokenStats">
				<span class="black">{tokenRecords[viewedTokenId].name} etoken data</span>
				<span>
					<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
						<path d="M2 2V21C2 21.55 2.45 22 3 22H22" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M18 5C18 6.15 17.9 7.19 17.71 8.14" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M8 17C10.28 16.88 14.12 15.62 16.32 11.88" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M6 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M12 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M18 22V20" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M2 5H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M2 11H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M2 17H4" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M19 14C20.1046 14 21 13.1046 21 12C21 10.8954 20.1046 10 19 10C17.8954 10 17 10.8954 17 12C17 13.1046 17.8954 14 19 14Z" fill="#AEB5BC"/>
						<path d="M17 12C18.1046 12 19 11.1046 19 10C19 8.89543 18.1046 8 17 8C15.8954 8 15 8.89543 15 10C15 11.1046 15.8954 12 17 12Z" stroke="#AEB5BC" stroke-width="2" stroke-linecap="round"/>
					</svg> <span>{tokenRecords[viewedTokenId].name} TX in the last 24 hours: <span class="black">{tokenInfo.pastDay}</span></span>
				</span>
				<span>
					<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
						<g clip-path="url(#clip0_10_516)">
						<path fill-rule="evenodd" clip-rule="evenodd" d="M4 3C4.55228 3 5 3.44772 5 4V23H24C24.5523 23 25 23.4477 25 24C25 24.5523 24.5523 25 24 25H4C3.44772 25 3 24.5523 3 24V4C3 3.44772 3.44772 3 4 3Z" fill="#F5F5F5"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M2 1C2.55228 1 3 1.44772 3 2V21H22C22.5523 21 23 21.4477 23 22C23 22.5523 22.5523 23 22 23H2C1.44772 23 1 22.5523 1 22V2C1 1.44772 1.44772 1 2 1Z" fill="#AEB5BC"/>
						<path d="M8 9C7.44772 9 7 9.44772 7 10V20C7 20.5523 7.44772 21 8 21H10C10.5523 21 11 20.5523 11 20V10C11 9.44772 10.5523 9 10 9H8Z" fill="#F5F5F5"/>
						<path d="M6 7C5.44772 7 5 7.44772 5 8V18C5 18.5523 5.44772 19 6 19H8C8.55228 19 9 18.5523 9 18V8C9 7.44772 8.55228 7 8 7H6Z" fill="#AEB5BC"/>
						<path d="M14 3C13.4477 3 13 3.44772 13 4V20C13 20.5523 13.4477 21 14 21H16C16.5523 21 17 20.5523 17 20V4C17 3.44772 16.5523 3 16 3H14Z" fill="#F5F5F5"/>
						<path d="M12 1C11.4477 1 11 1.44772 11 2V18C11 18.5523 11.4477 19 12 19H14C14.5523 19 15 18.5523 15 18V2C15 1.44772 14.5523 1 14 1H12Z" fill="#AEB5BC"/>
						<path d="M20 13C19.4477 13 19 13.4477 19 14V20C19 20.5523 19.4477 21 20 21H22C22.5523 21 23 20.5523 23 20V14C23 13.4477 22.5523 13 22 13H20Z" fill="#F5F5F5"/>
						<path d="M18 11C17.4477 11 17 11.4477 17 12V18C17 18.5523 17.4477 19 18 19H20C20.5523 19 21 18.5523 21 18V12C21 11.4477 20.5523 11 20 11H18Z" fill="#AEB5BC"/>
						</g>
						<defs>
						<clipPath id="clip0_10_516">
						<rect width="24" height="24" fill="white"/>
						</clipPath>
						</defs>
					</svg>
					<span>{tokenRecords[viewedTokenId].name} TX in total: <span class="black">{tokenInfo.total}</span></span>
				</span>
				<span>
					<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
						<path fill-rule="evenodd" clip-rule="evenodd" d="M4 21C3.44772 21 3 20.5523 3 20V14C3 13.4477 3.44772 13 4 13C4.55228 13 5 13.4477 5 14V20C5 20.5523 4.55228 21 4 21Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M8 21C7.44772 21 7 20.5523 7 20V9C7 8.44772 7.44772 8 8 8C8.55228 8 9 8.44772 9 9V20C9 20.5523 8.55228 21 8 21Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M12 21C11.4477 21 11 20.5523 11 20V12C11 11.4477 11.4477 11 12 11C12.5523 11 13 11.4477 13 12V20C13 20.5523 12.5523 21 12 21Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M16 21C15.4477 21 15 20.5523 15 20V17C15 16.4477 15.4477 16 16 16C16.5523 16 17 16.4477 17 17V20C17 20.5523 16.5523 21 16 21Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M20 21C19.4477 21 19 20.5523 19 20V12C19 11.4477 19.4477 11 20 11C20.5523 11 21 11.4477 21 12V20C21 20.5523 20.5523 21 20 21Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M4 8C5.10457 8 6 8.89543 6 10C6 11.1046 5.10457 12 4 12C2.89543 12 2 11.1046 2 10C2 8.89543 2.89543 8 4 8Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M8 3C9.10457 3 10 3.89543 10 5C10 6.10457 9.10457 7 8 7C6.89543 7 6 6.10457 6 5C6 3.89543 6.89543 3 8 3Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M12 6C13.1046 6 14 6.89543 14 8C14 9.10457 13.1046 10 12 10C10.8954 10 10 9.10457 10 8C10 6.89543 10.8954 6 12 6Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M20 6C21.1046 6 22 6.89543 22 8C22 9.10457 21.1046 10 20 10C18.8954 10 18 9.10457 18 8C18 6.89543 18.8954 6 20 6Z" fill="#AEB5BC"/>
						<path fill-rule="evenodd" clip-rule="evenodd" d="M16 11C17.1046 11 18 11.8954 18 13C18 14.1046 17.1046 15 16 15C14.8954 15 14 14.1046 14 13C14 11.8954 14.8954 11 16 11Z" fill="#AEB5BC"/>
					</svg>
					<span>Number of holder addresses: <span class="black">{tokenInfo.holders}</span></span>
				</span>

					
					
					
					
				
			</div>


			<div id="topHolders"><span class="black">Top 10 Addresses <button on:click={()=> etoken = !etoken} id={etoken ? 'etoken' : 'ecash'}><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" data-reactroot="">
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" fill="none" d="M22 12C22 6.47715 17.5228 2 12 2C6.47715 2 2 6.47715 2 12C2 17.5228 6.47715 22 12 22C17.5228 22 22 17.5228 22 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M7.5 12C7.5 11.4477 7.05228 11 6.5 11C5.94772 11 5.5 11.4477 5.5 12C5.5 12.5523 5.94772 13 6.5 13C7.05228 13 7.5 12.5523 7.5 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M13 12C13 11.4477 12.5523 11 12 11C11.4477 11 11 11.4477 11 12C11 12.5523 11.4477 13 12 13C12.5523 13 13 12.5523 13 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M18.5 12C18.5 11.4477 18.0523 11 17.5 11C16.9477 11 16.5 11.4477 16.5 12C16.5 12.5523 16.9477 13 17.5 13C18.0523 13 18.5 12.5523 18.5 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M12.5 12C12.5 11.7239 12.2761 11.5 12 11.5C11.7239 11.5 11.5 11.7239 11.5 12C11.5 12.2761 11.7239 12.5 12 12.5C12.2761 12.5 12.5 12.2761 12.5 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M7 12C7 11.7239 6.77614 11.5 6.5 11.5C6.22386 11.5 6 11.7239 6 12C6 12.2761 6.22386 12.5 6.5 12.5C6.77614 12.5 7 12.2761 7 12Z"/>
				<path stroke-linejoin="round" stroke-linecap="round" stroke-miterlimit="10" stroke-width="2" stroke="#221b38" d="M18 12C18 11.7239 17.7761 11.5 17.5 11.5C17.2239 11.5 17 11.7239 17 12C17 12.2761 17.2239 12.5 17.5 12.5C17.7761 12.5 18 12.2761 18 12Z"/>
				</svg>
			</button></span>
			{#each tokenInfo.topHolders as holder}
			<span class="holder">
				<a href={"https://explorer.e.cash/address/" + encode('etoken', decode(holder._id).type, decode(holder._id).hash)} target="_blank" rel="noreferrer">
				<span>
				<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
					<path fill-rule="evenodd" clip-rule="evenodd" d="M14.6352 2.43058C16.5552 0.52314 19.6448 0.52314 21.5648 2.43058L21.5671 2.43289C22.5232 3.38904 23 4.64722 23 5.9C23 7.15278 22.5232 8.41096 21.5671 9.36711L18.4871 12.4471C18.0966 12.8376 17.4634 12.8376 17.0729 12.4471C16.6823 12.0566 16.6823 11.4234 17.0729 11.0329L20.1529 7.95289C20.7167 7.38904 21 6.64722 21 5.9C21 5.15329 20.7171 4.41197 20.154 3.84827C19.0144 2.71737 17.186 2.71724 16.0463 3.84788L10.9525 8.95173C10.3825 9.51428 10.1 10.2536 10.1 11C10.1 11.1344 10.1102 11.2699 10.1289 11.4169C10.14 11.4623 10.1456 11.4981 10.1487 11.5198C10.1504 11.5316 10.1518 11.5428 10.153 11.5536C10.1824 11.6899 10.2162 11.81 10.2583 11.9243C10.2619 11.9341 10.2654 11.9439 10.2687 11.9538C10.3992 12.3455 10.6247 12.718 10.9489 13.0348L10.9571 13.0428C11.1245 13.2103 11.3006 13.3482 11.4745 13.4525C11.9481 13.7367 12.1016 14.3509 11.8175 14.8245C11.5333 15.2981 10.9191 15.4516 10.4455 15.1675C10.1209 14.9728 9.81846 14.7321 9.54692 14.4612C8.99721 13.9227 8.60576 13.282 8.37621 12.6009C8.28731 12.3569 8.22615 12.1199 8.17939 11.8861C8.17773 11.8778 8.17617 11.8694 8.17472 11.8611C8.16876 11.8348 8.16386 11.8082 8.16003 11.7814C8.12562 11.5406 8.09998 11.2811 8.09998 11C8.09998 9.74748 8.57666 8.48785 9.54506 7.5307L14.6352 2.43058ZM8.15998 11.69C8.15998 11.6922 8.15998 11.6944 8.16 11.6967L8.15998 11.69Z" fill="#AEB5BC"/>
					<path fill-rule="evenodd" clip-rule="evenodd" d="M12.1825 9.17551C12.4667 8.70192 13.0809 8.54836 13.5545 8.83251C13.8791 9.02724 14.1815 9.26791 14.4531 9.53885C15.0028 10.0774 15.3942 10.718 15.6238 11.3991C15.7127 11.6431 15.7738 11.8801 15.8206 12.1139C15.8222 12.1222 15.8238 12.1306 15.8253 12.1389C15.8312 12.1652 15.8361 12.1918 15.8399 12.2186C15.8744 12.4594 15.9 12.7189 15.9 13C15.9 14.2525 15.4233 15.5122 14.4549 16.4693L9.36479 21.5694C7.44481 23.4769 4.3552 23.4769 2.43522 21.5694L2.43289 21.5671C1.47674 20.611 1 19.3528 1 18.1C1 16.8472 1.47674 15.589 2.43289 14.6329L5.51289 11.5529C5.90342 11.1624 6.53658 11.1624 6.92711 11.5529C7.31763 11.9434 7.31763 12.5766 6.92711 12.9671L3.84711 16.0471C3.28326 16.611 3 17.3528 3 18.1C3 18.8467 3.28287 19.588 3.84595 20.1517C4.98554 21.2826 6.81395 21.2828 7.95368 20.1521C7.95419 20.1516 7.9547 20.1511 7.95522 20.1506L11.5319 16.5639L13.0475 15.0483C13.6175 14.4857 13.9 13.7464 13.9 13C13.9 12.8656 13.8897 12.7301 13.8711 12.5831C13.8599 12.5377 13.8544 12.5019 13.8513 12.4802C13.8496 12.4684 13.8482 12.4572 13.847 12.4464C13.8176 12.3101 13.7838 12.19 13.7417 12.0757C13.7381 12.0659 13.7346 12.0561 13.7313 12.0462C13.6007 11.6545 13.3752 11.282 13.0511 10.9652L13.0428 10.9572C12.8754 10.7897 12.6993 10.6518 12.5255 10.5475C12.0519 10.2633 11.8984 9.64909 12.1825 9.17551ZM15.84 12.31C15.84 12.3078 15.84 12.3056 15.84 12.3033V12.31Z" fill="#AEB5BC"/>
				</svg>
				<span class="black">{encode((etoken ? 'etoken' : 'ecash'), decode(holder._id).type, decode(holder._id).hash).slice(0,15)}...{encode((etoken ? 'etoken' : 'ecash'), decode(holder._id).type, decode(holder._id).hash).slice(-8)}</span>
				</span></a>
				<span><span class="black">{Number(holder.token_balance).toLocaleString()}</span>&nbsp;<span>{tokenRecords[viewedTokenId].ticker}</span></span>
			</span>
			{/each}
			</div>
		</div>
	</div>
	{/await}
{/if}
</main>

<style>
	#ecash, #etoken{
		height: 24px;
		width: 24px;
		padding: 0;
		border: none;
		margin: 0;
		margin-right: 15px;
		border-radius: 20px;
		place-items: center;
	}

	#ecash svg, #etoken svg{
		margin: 0;
	}

	#ecash{
		background-color: #FFE9C8;
	}

	#etoken{
		background-color: #EAC9F7;
	}


	#logo img{
		height: 25px;
		width: 25px;
	}

	#logo button{
		padding: 0;
		margin: 0;
		display: flex;
		align-items: center;
	}


	input{
	font-weight: 600;
	font-size: 24px;
	width: 0px;

	}
	input#tokenid::placeholder {
    color: #AEB5BC;
	}

	#topHolders{
		max-height: 30vh;
		overflow-y: scroll;
		overflow-x: hidden;
	}

	#topHolders > span{
		justify-content: space-between;
		width: 100%;
	}

	#topHolders > span >a + span{
		margin-right: 10px;
	}

	#topHolders > span{
		margin: 7px 10px;
	}

	#topHolders svg{
		margin-right: 5px;
	}

	#topHolders span{
		display: inline-flex;
		align-items: center;
	}

	#tokenStats, #topHolders{
		padding: 20px;
		align-items: flex-start;
	}

	#tokenStats > span{
		display: inline-flex;
		align-items: center;
		margin: 10px 10px;
	}

	#tokenStats > span > svg + span{
		margin-left: 5px;
	}

	#basicInfo a{
		color: rgba(174, 181, 188, 1);
		text-decoration: underline;
	}

	#basicInfo{
		align-items: flex-start;
		padding: 15px;
	}

	#basicInfo > div{
		display:flex;
		flex-direction: column;
	}

	#basicInfo > span{
		margin-left: 10px;
		margin-top: 10px;
	}

	#basicInfo > div > span{
		display: inline-flex;
		align-items: center;
		margin: 7px 5px;
	}

	.push{
		margin-left: 10px !important;
	}

	#basicInfo > div > span > span{
		margin-left: 5px;
	}

	#tokenName > span{
		margin-left: -5px !important;
	}

	#recentTransactions{
		max-height: 30vh;
		overflow: scroll;
		margin: 0;
	}

	#recentContainer > span{
		text-align: left;
		margin-left: 10px;
		padding: 10px;
	}

	.recentTransaction div{
		border-radius: 15px;
		padding: 10px;
	}

	.recentTransaction div{
		display: flex;
		flex-direction: column;
	}

	.recentTransaction div span{
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin: 2px 0;
		color: rgba(174, 181, 188, 1);
	}

	.recentTransaction svg{
		margin-right: 5px;
	}

	.recentTransaction div span svg + span{
		flex-grow: 1;

	}

	div{
		margin: 5px;
		padding: 5px 5px 10px 5px;
	}

	.recentTransaction div{
		margin: 2px 5px;
	}

	main{
		max-width: 120vh;
	}

	#back svg{
		height:20px;
		width: 20px;
	}

	#infoContainer{
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: 0;
		padding-top: 0;

		margin-top: 10px;
	}

	#infoContainer > div{
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
	}

	#infoContainer > div{
		width: 100%;
		margin: 0;
		padding: 0;
	}

	#infoContainer > div > div{
		background-color: rgb(255,255,255);
		box-shadow: 2px 4px 10px rgba(0, 0, 0, 0.1);
		border-radius: 20px;
		flex-grow: 1;
		display: flex;
		flex-direction: column;
	}

	.tick, .black, button:hover, #search, #github{
		color:rgba(47, 47, 47, 1) !important;
	}
	
	#search{
		width: 52px;
		color: #aeb5bc;
		background-color: #f4f4f4ab;
		outline: none;
		border-radius: 200px;
		border: none;
	}

	#credit{
		position: relative;
    margin-top: 30px;
    left: 0px;
    right: 0px;
    display: inline-flex;
    justify-content: center;
	color:#d2d2d2;
    width: 100%;
	}
	#tecto{
		color: rgb(10,74,156);
	}
	#logo{
		height: 40px;
		width: 40px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	#github a{
		color: #2F2F2F;
	}

	#github, #back{
		display: flex;
		align-items: center;
		justify-content: center;
	}

	#github >*{
		margin: 0 10px;
	}

	#github svg{
		margin-right: 0px;
	}

	#iconsbar{
		display:flex;
		justify-content: space-between;
		margin: 10px 10px;
		margin-bottom: 0;
	}

	#iconsbar > span, #iconsbar > button{
		background-color: #FFFFFF;
		border-radius: 20px;
		box-shadow: 2px 4px 10px rgba(0, 0, 0, 0.1);
	}

	#logo{
		box-shadow:(2px 4px 100px rgba(0, 0, 0, 0.15));
	}

	#iconsbar button{
		border:none;
	}


	.name{
		flex-grow: 1;
		text-align: left;
	}

	.tokenid{
		margin-right: 10px;
	}

	.icon span{
			display: grid;
			display: flex;
	}

    .icon span *{
        grid-column: 1;
        grid-row: 1;
    }

    .icon{
        flex-shrink: 0;
        font-size: 32px;
        width: 30px;
        height: 30px;
        background-color: #FFFFFF;
        padding: 10px;
        border-radius: 40px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        margin-right: 10px;
        overflow:hidden
        

    }
    .icon span{
        margin: auto;

    }

    .tick{
        z-index: 1;
    }

    .icon img{
        z-index: 2;
        margin: auto;
        width: 50px;
        height: 50px;
        position: relative;
        right: 10px;
    }
	.small img{
		width:32px;
		height:32px;
	}
	.small{
		width:40px;
		height:32px;
		font-size: 18px;
		margin: 0;
		padding: 0 10px;
		padding-right: 0px;
		margin-left: 0px;
	}
	.small > span> span{
		margin-left: -19px;
	}

	.small > span > img + span{
		margin-left:auto;
	}

	.small > span > img{
		margin-left: -5px;
	}

    .icon img + span{
        opacity: 0;
		display:none;
    }


	#recommended, #data{
		margin-left: 15px;
	}

	#newTokens, #eTokenVolume{
		margin-left: 20px;
		display: inline-flex;
		align-items: center;
		margin-top: 10px;

	}

	#newTokens svg, #eTokenVolume svg{
		height: 20px;
		width: 20px;
	}

	#eTokenVolume span{
		display:inline-flex;
		align-items:center;
	}

	#container{
		border-radius: 20px;
		box-shadow: 2px 4px 50px rgba(0, 0, 0, 0.1);
		background-color: #FFFFFF;
		padding: 20px 10px;
		display:flex;
		flex-direction: column;
		margin-top: 15px;
	}

	#container > span{
		text-align: left;
	}

	#featuredTokens{
		display:flex;
		flex-direction: column;
		max-height: 48vh;
		overflow: scroll;
		margin: 10px 0px;
	}

	.featuredToken{
		display:inline-flex;
		align-items: center;
		flex-wrap: wrap;
		border:none;
		border-radius: 20px;
	}

	.featuredToken:hover, .recentTransaction div:hover{
		background-color: rgba(244, 245, 246, 1);
	}

	#input{
		width: 100%;
		display:flex;
		box-sizing: border-box;
		padding: 0 10px;
		align-self: center;
		margin-bottom: 10px;
	}

	#input input{
		flex-grow: 1;
		border: none;
	}

	#input input:focus{
		outline: none;
	}

	main {
		text-align: center;
		padding: 0;
		margin: 0 auto;
		word-break: break-word;
	}

	#search:hover, #github:hover, button#back:hover, #logo:hover{
		transition: .8s;
		background: linear-gradient(179.05deg, #CCE0FF 0.81%, #ECF4FF 0.82%, rgba(235, 243, 255, 0.916667) 75.23%, rgba(201, 213, 255, 0) 109.05%);
	}



</style>
