<script>
	import axios from 'axios';
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import * as topojson from 'topojson-client';
	import { interpolateReds } from 'd3-scale-chromatic';
	import { contourDensity } from 'd3-contour';

	let prefecture = 'kanagawa'; // デフォルトの都道府県
	let model = 'v0a'; // モデルを v0a に固定
	let oxData = {}; // 初期値を空のオブジェクトに変更
	let loading = false;
	let error = null;
	let selectedDateTime = 'now'; // デフォルトは現在時刻
	let ptable = null; // 確率表を保持する状態変数
	let ptableError = null; // 確率表取得時のエラーを保持する状態変数
	let ptableLoading = false; // 確率表取得中のローディング状態
	let ptableRows = []; // 確率表の行データを保持する変数
	let ptableColumns = []; // 確率表のカラムデータを保持する変数
	const zValue = 120; // z の値を 120 に固定
	let addresses = {}; // 住所データを保持する変数
	let mapData;
	let oxydantData = [];
	let currentHour = 1;
	let isPlaying = false;
	let animationInterval;
	let addressCache = new Map(); // 住所のキャッシュ
	let methaneData = {}; // メタンデータを保持する状態変数
	let methaneLoading = false; // メタンデータ取得中のローディング状態
	let methaneError = null; // メタンデータ取得時のエラーを保持する状態変数
	let methaneMapData = []; // メタンデータの地図表示用データ

	// 日時を ISO 8601 形式に変換する関数 (JST に対応)
	function getISO8601(date) {
		const jstOffset = 9 * 60; // JST は UTC+9 なので、9時間分のオフセットを分単位で設定
		const utc = date.getTime() + (date.getTimezoneOffset() * 60000); // UTC に変換
		const jst = new Date(utc + (jstOffset * 60000)); // JST に変換

		const year = jst.getFullYear();
		const month = (jst.getMonth() + 1).toString().padStart(2, '0'); // 月は 0-11 なので +1 する
		const day = jst.getDate().toString().padStart(2, '0');
		const hour = jst.getHours().toString().padStart(2, '0');
		const minute = jst.getMinutes().toString().padStart(2, '0');

		return `${year}-${month}-${day}T${hour}:${minute}:00+09:00`;
	}

	// データを取得する関数
	async function fetchData() {
		loading = true;
		error = null;
		oxData = {};

		try {
			const formattedDatehour = selectedDateTime === 'now' 
				? getISO8601(new Date())
				: selectedDateTime;

			const response = await axios.get(
				`/api/ox/${model}/${prefecture}/${formattedDatehour}`
			);

			oxData = response.data.data;
			console.log('oxDataの内容:', oxData);
			oxydantData = loadOxydantData();
			console.log('oxydantDataの内容:', oxydantData);

			if (oxData && oxData.lon && oxData.lat) {
				await fetchAddresses();
			}

		} catch (err) {
			error = err.response
				? `データ取得エラー: ${err.response.status} - ${err.response.statusText}`
				: err.request
					? 'データ取得エラー: サーバーからの応答がありません。'
					: 'データ取得エラー: リクエストを送信できませんでした。';
		} finally {
			loading = false;
		}
	}

	// 確率表を取得する関数
	async function fetchPtable() {
		ptableLoading = true;
		ptableError = null;
		try {
			const response = await axios.get(
				`/api/ptable/${model}`
			);
			// JSON データを解析して、行と列のデータに変換
			const jsonData = response.data;
			ptableColumns = Object.keys(jsonData);
			const indexKeys = new Set();
			ptableColumns.forEach(col => {
				Object.keys(jsonData[col]).forEach(key => indexKeys.add(key));
			});
			ptableRows = Array.from(indexKeys).map(index => {
				const [index1, index2] = index.slice(1, -1).split(',').map(Number); // 修正箇所
				const row = { index1, index2 };
				ptableColumns.forEach(col => {
					row[col] = jsonData[col][index]; // 修正箇所
				});
				return row;
			});
			ptable = response.data;

		} catch (err) {
			if (err.response) {
				ptableError = `確率表取得エラー: ${err.response.status} - ${err.response.statusText}`;
			} else if (err.request) {
				ptableError = '確率表取得エラー: サーバーからの応答がありません。';
			} else {
				ptableError = '確率表取得エラー: リクエストを送信できませんでした。';
			}
		} finally {
			ptableLoading = false;
		}
	}

	// 住所取得のバッチ処理
	async function fetchAddresses() {
		if (!oxData || !oxData.lon || !oxData.lat) return;

		const batchSize = 5;
		const totalPoints = oxData.lon.length;
		
		for (let i = 0; i < totalPoints; i += batchSize) {
			const batch = Array.from(
				{ length: Math.min(batchSize, totalPoints - i) },
				(_, j) => ({
					index: i + j,
					lon: oxData.lon[i + j],
					lat: oxData.lat[i + j]
				})
			);

			await Promise.all(
				batch.map(async ({ index, lon, lat }) => {
					const address = await getAddress(lat, lon);
					addresses[index] = address;
				})
			);

			await new Promise(resolve => setTimeout(resolve, 500));
		}
	}

	// 住所取得関数
	async function getAddress(lat, lng) {
		const cacheKey = `${lat},${lng}`;
		if (addressCache.has(cacheKey)) {
			return addressCache.get(cacheKey);
		}

		try {
			const response = await fetch(
				`https://mreversegeocoder.gsi.go.jp/reverse-geocoder/LonLatToAddress?lat=${lat}&lon=${lng}`
			);
			const data = await response.json();

			let address = `${lat}, ${lng}`;
			if (data.results && data.results.muniCd) {
				address = `${data.results.lv01Nm || ''}${data.results.lv02Nm || ''}${data.results.lv03Nm || ''}`;
			}
			
			addressCache.set(cacheKey, address);
			return address;
		} catch (error) {
			console.error('住所取得エラー:', error);
			return `${lat}, ${lng}`;
		}
	}

	// オキシダントデータの読み込み関数
	function loadOxydantData() {
		if (!oxData || !oxData.lon || !oxData.lat) return [];
		
		return oxData.lon.flatMap((lon, i) => 
			Array.from({ length: 24 }, (_, hour) => {
				const value = oxData[`+${hour + 1}`]?.[i];
				return value !== undefined ? {
					longitude: lon,
					latitude: oxData.lat[i],
					hour: hour + 1,
					value: value
				} : null;
			}).filter(Boolean)
		);
	}

	// アニメーションの開始/停止
	function toggleAnimation() {
		isPlaying = !isPlaying;
		if (isPlaying) {
			animationInterval = setInterval(async () => {
				currentHour = (currentHour % 24) + 1;
				await fetchData();
				drawMap();
			}, 1000);
		} else {
			clearInterval(animationInterval);
		}
	}

	// メタンデータを取得する関数
	async function fetchMethaneData() {
		methaneLoading = true;
		methaneError = null;
		methaneData = {};

		try {
			const formattedDatehour = selectedDateTime === 'now' 
				? getISO8601(new Date())
				: selectedDateTime;

			const response = await axios.get(
				`/api/methane/${model}/${prefecture}/${formattedDatehour}`
			);

			methaneData = response.data.data;
			console.log('methaneDataの内容:', methaneData);

		} catch (err) {
			methaneError = err.response
				? `メタンデータ取得エラー: ${err.response.status} - ${err.response.statusText}`
				: err.request
					? 'メタンデータ取得エラー: サーバーからの応答がありません。'
					: 'メタンデータ取得エラー: リクエストを送信できませんでした。';
		} finally {
			methaneLoading = false;
		}
	}

	// メタンデータを地図表示用に変換する関数
	function loadMethaneMapData() {
		if (!methaneData || !methaneData.lon || !methaneData.lat) return [];
		
		return methaneData.lon.map((lon, i) => ({
			longitude: lon,
			latitude: methaneData.lat[i],
			value: methaneData.value?.[i] || 0
		}));
	}

	// 初期データ取得
	onMount(async () => {
		try {
			// 地図データの取得
			const response = await fetch('https://raw.githubusercontent.com/dataofjapan/land/master/japan.topojson');
			if (!response.ok) {
				throw new Error('地図データの取得に失敗しました');
			}
			const topoData = await response.json();
			mapData = topojson.feature(topoData, topoData.objects.japan);
			
			// 神奈川県のフィーチャーを抽出
			const kanagawaFeature = mapData.features.find(f => 
				f.properties && f.properties.nam_ja && f.properties.nam_ja.includes('神奈川県')
			);
			
			if (kanagawaFeature) {
				mapData = {
					type: 'FeatureCollection',
					features: [kanagawaFeature]
				};
			}

			console.log('地図データ読み込み完了:', mapData);

			// データの取得と描画
			await Promise.all([fetchData(), fetchPtable(), fetchMethaneData()]);
			drawMap();
		} catch (err) {
			console.error('初期化エラー:', err);
			error = `初期化エラー: ${err.message}`;
			mapData = {
				type: 'FeatureCollection',
				features: []
			};
		}
	});

	// 都道府県変更時の処理
	function handlePrefectureChange(event) {
		prefecture = event.target.value;
		fetchData();
	}

	// 時刻変更時の処理
	function handleDateTimeChange(event) {
		selectedDateTime = event.target.value;
		fetchData();
	}

	// ptable から確率を取得する関数
	function getProbability(x, y) {
		if (!ptable) return null; // ptable が null の場合は null を返す
		if (!ptableRows || ptableRows.length === 0) return null;
		const row = ptableRows.find(row => Math.floor(x / 5) * 5 === row.index1 && row.index2 === y); // 修正箇所
		if (!row || row[zValue] === undefined) return null; // 修正箇所
		return Math.round(row[zValue] * 100);
	}

	// 確率に応じた背景色を返す関数
	function getBackgroundColor(ptable, oxData, hour, i) {
		if (!ptable) return 'white'; // ptable が null の場合は white を返す
		const probability = getProbability(Math.round(oxData[`+${hour}`][i]), hour);

		if (probability === null) return 'white'; // null の場合は white を返す

		let hue, saturation, lightness;

		if (probability <= 25) {
			// 0% - 25%: 水色から灰色へ
			hue = 180; // 水色
			saturation = 100 - (probability / 25) * 100; // 100% -> 0%
			lightness = 80 - (probability / 25) * 30; // 80% -> 50%
		} else if (probability <= 50) {
			// 25% - 50%: 灰色から赤色へ
			hue = 0; // 赤色
			saturation = ( (probability - 25) / 25) * 100; // 0% -> 100%
			lightness = 50 + ((probability - 25) / 25) * 0; // 50% -> 50%
		} else {
			// 50% 以上: 赤色
			hue = 0;
			saturation = 100;
			lightness = 50;
		}

		return `hsl(${hue}, ${saturation}%, ${lightness}%)`;
	}

	// 地図の描画
	async function drawMap() {
		const container = document.getElementById('map-container');
		if (!container) return;

		container.innerHTML = '';
		const width = 800;
		const height = 600;

		const svg = d3.select(container)
			.append('svg')
			.attr('width', width)
			.attr('height', height);

		// 投影法の設定（神奈川県中心）
		const projection = d3.geoMercator()
			.center([139.3, 35.4])
			.scale(20000)
			.translate([width / 2, height / 2]);

		const path = d3.geoPath().projection(projection);

		// 神奈川県の地図を描画
		svg.append('g')
			.selectAll('path')
			.data(mapData.features)
			.enter()
			.append('path')
			.attr('d', path)
			.attr('fill', '#f0f0f0')
			.attr('stroke', '#999')
			.attr('stroke-width', '1px');

		// メタンデータの等高線を描画
		methaneMapData = loadMethaneMapData();
		if (methaneMapData.length > 0) {
			const points = methaneMapData.map(d => {
				const projected = projection([d.longitude, d.latitude]);
				return projected ? [projected[0], projected[1], d.value] : null;
			}).filter(Boolean);

			if (points.length > 0) {
				const maxValue = d3.max(points, d => d[2]);
				const colorScale = d3.scaleSequential(interpolateReds).domain([0, maxValue]);

				const contours = d3.contourDensity()
					.x(d => d[0])
					.y(d => d[1])
					.weight(d => d[2])
					.size([width, height])
					.bandwidth(20)
					.thresholds(10)
					(points);

				svg.append('g')
					.selectAll('path')
					.data(contours)
					.enter()
					.append('path')
					.attr('d', d => {
						if (!d.coordinates?.[0]?.[0]) return '';
						return d3.line().curve(d3.curveBasisClosed)(d.coordinates[0][0]);
					})
					.attr('fill', (d, i) => colorScale(i / contours.length * maxValue))
					.attr('opacity', 0.6)
					.attr('stroke', '#fff')
					.attr('stroke-width', 0.5);

				// メタン濃度の凡例
				const legendWidth = 200;
				const legendHeight = 20;
				const legend = svg.append('g')
					.attr('transform', `translate(${width - legendWidth - 20}, 20)`);

				const gradient = legend.append('defs')
					.append('linearGradient')
					.attr('id', 'methane-gradient')
					.attr('x1', '0%').attr('y1', '0%')
					.attr('x2', '100%').attr('y2', '0%');

				gradient.selectAll('stop')
					.data(colorScale.ticks(5))
					.enter()
					.append('stop')
					.attr('offset', d => `${(d / maxValue) * 100}%`)
					.attr('stop-color', d => colorScale(d));

				legend.append('rect')
					.attr('width', legendWidth)
					.attr('height', legendHeight)
					.style('fill', 'url(#methane-gradient)');

				legend.selectAll('text')
					.data(colorScale.ticks(5))
					.enter()
					.append('text')
					.attr('x', d => (d / maxValue) * legendWidth)
					.attr('y', legendHeight + 15)
					.text(d => Math.round(d))
					.style('font-size', '10px')
					.style('text-anchor', 'middle');
			}
		}

		// オキシダントデータの等高線を描画
		const hourData = oxydantData.filter(d => d.hour === currentHour);
		if (hourData.length > 0) {
			const points = hourData.map(d => {
				const projected = projection([d.longitude, d.latitude]);
				return projected ? [projected[0], projected[1], d.value] : null;
			}).filter(Boolean);

			if (points.length > 0) {
				const maxValue = d3.max(points, d => d[2]);
				const colorScale = d3.scaleSequential(interpolateReds).domain([0, maxValue]);

				const contours = d3.contourDensity()
					.x(d => d[0])
					.y(d => d[1])
					.weight(d => d[2])
					.size([width, height])
					.bandwidth(20)
					.thresholds(10)
					(points);

				svg.append('g')
					.selectAll('path')
					.data(contours)
					.enter()
					.append('path')
					.attr('d', d => {
						if (!d.coordinates?.[0]?.[0]) return '';
						return d3.line().curve(d3.curveBasisClosed)(d.coordinates[0][0]);
					})
					.attr('fill', (d, i) => colorScale(i / contours.length * maxValue))
					.attr('opacity', 0.6)
					.attr('stroke', '#fff')
					.attr('stroke-width', 0.5);

				// オキシダントの凡例
				const legendWidth = 200;
				const legendHeight = 20;
				const legend = svg.append('g')
					.attr('transform', `translate(${width - legendWidth - 20}, 60)`);

				const gradient = legend.append('defs')
					.append('linearGradient')
					.attr('id', 'ox-gradient')
					.attr('x1', '0%').attr('y1', '0%')
					.attr('x2', '100%').attr('y2', '0%');

				gradient.selectAll('stop')
					.data(colorScale.ticks(5))
					.enter()
					.append('stop')
					.attr('offset', d => `${(d / maxValue) * 100}%`)
					.attr('stop-color', d => colorScale(d));

				legend.append('rect')
					.attr('width', legendWidth)
					.attr('height', legendHeight)
					.style('fill', 'url(#ox-gradient)');

				legend.selectAll('text')
					.data(colorScale.ticks(5))
					.enter()
					.append('text')
					.attr('x', d => (d / maxValue) * legendWidth)
					.attr('y', legendHeight + 15)
					.text(d => Math.round(d))
					.style('font-size', '10px')
					.style('text-anchor', 'middle');
			}
		}
	}
</script>

<main>
	<h1>光化学オキシダント予測</h1>

	<div class="controls">
		<div>
			<label for="prefecture">都道府県:</label>
			<select id="prefecture" bind:value={prefecture} on:change={handlePrefectureChange}>
				<option value="kanagawa">神奈川県</option>
			</select>
		</div>

		<div>
			<label for="dateTime">時刻:</label>
			<select id="dateTime" bind:value={selectedDateTime} on:change={handleDateTimeChange}>
				<option value="now">現在時刻</option>
				<option value="2015-07-27T06:00:00+09:00">2015年7月27日午前6時</option>
				<option value="2011-08-13T06:00:00+09:00">2011年8月13日午前6時</option>
			</select>
		</div>
	</div>

	{#if loading || methaneLoading}
		<p>データを読み込み中です...</p>
	{:else if error || methaneError}
		<p style="color: red;">エラー: {error || methaneError}</p>
	{:else}
		{#if methaneData && methaneData.lon && methaneData.lat}
			<div class="methane-section">
				<h2>メタン濃度</h2>
				<table>
					<thead>
						<tr>
							<th>経度</th>
							<th>緯度</th>
							<th>住所</th>
							<th>メタン濃度 (ppb)</th>
						</tr>
					</thead>
					<tbody>
						{#each methaneData.lon as lon, i}
							<tr>
								<td>{lon.toFixed(2)}</td>
								<td>{methaneData.lat[i].toFixed(2)}</td>
								<td>{addresses[i] || '住所取得中...'}</td>
								<td>{methaneData.value?.[i]?.toFixed(2) || '-'}</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		{/if}

		{#if oxData && oxData.XY && oxData.lon && oxData.lat}
			<div class="ox-section">
				<h2>光化学オキシダント</h2>
				<table>
					<thead>
						<tr>
							<th>XY</th>
							<th>経度</th>
							<th>緯度</th>
							<th>住所</th>
							{#each Array.from({ length: 24 }, (_, i) => i + 1) as hour}
								<th>+{hour}</th>
							{/each}
						</tr>
					</thead>
					<tbody>
						{#each oxData.XY as item, i}
							<tr>
								<td>{oxData.XY[i]}</td>
								<td>{oxData.lon[i].toFixed(2)}</td>
								<td>{oxData.lat[i].toFixed(2)}</td>
								<td>{addresses[i] || '住所取得中...'}</td>
								{#each Array.from({ length: 24 }, (_, j) => j + 1) as hour}
									<td
										class:highlight={Math.round(oxData[`+${hour}`][i]) >= 120}
										style="background-color: {getBackgroundColor(ptable, oxData, hour, i)};"
									>
										{Math.round(oxData[`+${hour}`][i])}
										{#if ptable}
											<br />
											{getProbability(Math.round(oxData[`+${hour}`][i]), hour) ?? '-'}%
										{/if}
									</td>
								{/each}
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		{:else}
			<p>データがありません。</p>
		{/if}
	{/if}
</main>

<div class="container">
	<h2>予測値マップ</h2>
	<div id="map-container"></div>
	<div class="controls">
		<button on:click={toggleAnimation}>{isPlaying ? '停止' : '再生'}</button>
		<span>現在の時間: {currentHour}時</span>
	</div>
</div>

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 100%;
		margin: 0 auto;
	}

	.controls {
		display: flex;
		gap: 20px;
		justify-content: center;
		margin-bottom: 20px;
	}

	table {
		width: 100%;
		border-collapse: collapse;
		margin-top: 1em;
	}

	th, td {
		border: 1px solid #ccc;
		padding: 0.5em;
		text-align: center;
	}

	.highlight {
		color: green;
		font-weight: bold;
	}

	.container {
		max-width: 800px;
		margin: 0 auto;
		padding: 20px;
	}

	#map-container {
		border: 1px solid #ccc;
		margin-bottom: 20px;
	}

	button {
		padding: 8px 16px;
		background-color: #4CAF50;
		color: white;
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	button:hover {
		background-color: #45a049;
	}

	.methane-section, .ox-section {
		margin-top: 2em;
	}

	.methane-section h2, .ox-section h2 {
		font-size: 1.5em;
		margin-bottom: 1em;
		color: #333;
	}
</style>