<script>
	const numFormat = new Intl.NumberFormat("en-US", {minimumFractionDigits: 2, maximumFractionDigits: 2,signDisplay:"auto"});
	const intFormat = new Intl.NumberFormat("en-US", {minimumFractionDigits: 0, maximumFractionDigits: 0,signDisplay:"auto"});
	import SVGCanvas from './SVGCanvas.svelte'
	

	
	let zoomFactor = 1
	const zoomBase = 50;

	let hues = [60,20,100]


	let layers = [
	]
	
	
	$: zoom = Math.exp(zoomFactor) * zoomBase

	let press = {
		x:0,
		y:0,
		initialx: 0,
		initialy: 0,
	}
	let pressOffset = false
	let pressScale = false
	let pressLayer = null		
	let xdominant = true



	function reset(l) {
		layers[l].scale.x = 1
		layers[l].scale.y = 1
		layers[l].offset.x = 0
		layers[l].offset.y = 0
		layers[l].baseParam.integer = true
		layers[l].baseParam.value = 2
		layers[l].onlyPositives = false
	}

	function removeLayer(l) {
		hues.push(layers[l].hue)
		hues = hues
		layers = [...layers.slice(0, l), ...layers.slice(l+1)]
	}

	function addLayer() {
		const hue = hues.pop()
		
		hues = hues

		if(hue) {
			layers = [...layers, {
				hue: hue,
				scale: {x:1,y:1},
	 			offset: {x:0,y:0},
	 			baseParam: {
	 				intger: true,
	 				value: 2,
	 				get castValue() { return this.integer ? Math.round(this.value) : this.value }
	 			},
	 			onlyPositives: false,
	 			selectedFunction: 3,
			}]
		}
	}
	
	function onMouseDown({detail: local}) {
		if(!local.target.getAttribute('data-control')) {
			return
		}
		press.x = local.x
		press.y = local.y

		const control = local.target.getAttribute('data-control')
		pressLayer = parseInt(local.target.getAttribute('data-layer'), 10)
		pressOffset = control === 'offset'
		pressScale = control === 'scale'
		xdominant = true


		if(pressOffset) {
			press.initialx = layers[pressLayer].offset.x
			press.initialy = layers[pressLayer].offset.y
		} else if(pressScale) {
			press.initialx = layers[pressLayer].scale.x
			press.initialy = layers[pressLayer].scale.y
		}
	}

	function onMouseUp(evt) {
		 pressOffset = false
		 pressScale = false
		 pressLayer = null
	}

	function onMouseMove({detail: local}) {
		const dx = local.x - press.x
		const dy = local.y - press.y
		const d2 = (dx*dx + dy*dy)/zoom
		
		if(d2 > 50 && Math.abs(dx) * 8 < Math.abs(dy)) {
			xdominant = false
		} else if(d2 > 50 && Math.abs(dx) > 8 * Math.abs(dy)) {
			xdominant = true
		}

		if(pressOffset) {
			if(local.shift) {
				if(local.alt ^ xdominant) {
					layers[pressLayer].offset.x = press.initialx + dx/zoom
					layers[pressLayer].offset.y = press.initialy
				} else {
					layers[pressLayer].offset.y = press.initialy - dy/zoom
					layers[pressLayer].offset.x = press.initialx
				}
			} else {
				layers[pressLayer].offset.x = press.initialx + dx/zoom
				layers[pressLayer].offset.y = press.initialy - dy/zoom
			}
		} else if(pressScale) {
			if(local.shift) {			
				if(local.alt ^ xdominant) {
					layers[pressLayer].scale.x = press.initialx + dx/zoom
					layers[pressLayer].scale.y = press.initialy
				} else {
					layers[pressLayer].scale.y = press.initialy - dy/zoom
					layers[pressLayer].scale.x = press.initialx
				}
			} else {
				layers[pressLayer].scale.x = press.initialx + dx/zoom
				layers[pressLayer].scale.y = press.initialy - dy/zoom
			}
		}
	}

	function segments(resolution, min, max) {
		return Array(Math.round(Math.abs(resolution))).fill(null).map((_,i,all) => min + (i/resolution)*(max-min))
	}
	function fixedSegments(mincount, maxcount, resolution, min, max) {
		const count = Math.ceil(Math.abs((max-min)/resolution))
		const factor = Math.min(Math.max(mincount, count), maxcount)/Math.max(1, count)
		return Array(Math.round(count*factor)).fill(null).map((_,i,all) => min/factor + Math.sign(max-min)*i*resolution/factor)
	}


	
	
	let functions = [
		{supportsScale:(s) => true, hue: 0, name: 'constant: 1', fractionalB: (bIsInt, x) => true, fn:(x, b) => 1},
		{supportsScale:(s) => true, hue: 50, name: 'linear: x', fractionalB: (bIsInt, x) => true, fn:(x, b) => x},
		{supportsScale:(s) => true, hue: 100, name: 'reciprocal: 1/x^b', fractionalB: (bIsInt, x) => bIsInt || x>0, fn:(x, b, sign) => x=== 0 ? (sign<0?(Infinity*Math.pow(-1,Math.ceil(b))):Infinity) : 1/Math.pow(x,b)},
		{supportsScale:(s) => true, hue: 150, name: 'power: x^b', fractionalB: (bIsInt, x) => bIsInt || x>=0, fn:(x, b) => Math.pow(x, b)},
		{supportsScale:(s) => true, hue: 200, name: 'root: x^(1/b)', fractionalB: (bIsInt, x) => bIsInt || x>=0, fn:(x, b) => Math.pow(x, 1/b)},
		{supportsScale:(s) => true, hue: 250, name: 'exponential: b^x', fractionalB: (bIsInt, x) => true, fn:(x, b) => Math.pow(b, x)},
		{supportsScale:(s) => true, hue: 300, name: 'logarithmic: log_b(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => x >0 && b>0 ? Math.log(x)/Math.log(b) : (x<0?undefined:(b<1?1:-1)*Infinity)},
		{supportsScale:(s) => true, hue: 340, name: 'log-linear: x⋅log_b(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => x >0 && b>0 ? x*Math.log(x)/Math.log(b) : (x<0?undefined:0)},
		{supportsScale:(s) => true, hue: 30, name: 'sinusoidal: sin(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => Math.sin(x)},
	];

	function clamp(v, min, max) {
		return Math.max(min, Math.min(max, v))
	}

	function onWheel(evt) {
		zoomFactor = clamp(zoomFactor + (-(evt.deltaY)/120/3), -5, 5)
	}

	function layerColor(layers, l) {
		const layer = layers[l]
		return `hsl(${intFormat.format(-30-layer.hue)}, 100%, 49%)`
	}
</script>

<style>
	:global(body, html) {
		margin: 0;
		padding: 0;
	}

	.controls {
		background: #fff;
		padding: 1em;
		width: 100%;
		overflow: auto;
		border-right: 0.3em solid #eee;
		box-sizing: border-box;
	}

	.checkbox-list {
		display: flex;
		flex-direction: column;
		gap: 0.5em;
		padding: 0.2em;
	}

	.checkbox-row {
		display: flex;
		flex-direction: row;
		gap: 0.5em;
		white-space: nowrap;
		align-items: center;
	}

	.checkbox-list > label {
		cursor: pointer;
	}

	button {
		cursor: pointer;
	}

	h1 {
		margin: 0 0 1em;
		line-height: 1;
		text-align: center;
	}

	[data-control] {
		cursor: move;
		pointer-events: all;
	}

	.container {
		position: absolute;
		inset: 0;
		display: grid;
		grid-template-columns: 25em 1fr;
	}

	.zoom-control {
		display: flex;
		align-items: center;
		gap: 1em;
		position: absolute;
		right: 0;
		bottom: 0;
		padding: 1em;
		z-index: 100;
	}

	input[type=range] {
		margin: 0;
		padding: 0;
		width: 100%;
		box-sizing: border-box;
	}

	.layer-box {
		margin: 0.2em;
		border: 3px solid gray;
		padding: 0.5em;
		border-color: var(--layer-color, #333);
	}

	h3 {
		margin: 0;
		background: var(--layer-color, #333);
		color: #fff;
		padding: 0.5em;
		display: flex;
		justify-content: space-between;
		align-items: center;
		font-size: 1em;
	}

	h3 button {
		font: inherit;
		font-size: 0.8em;
		margin: 0;
		padding: 0.1em 1em;
		font-weight: normal;
		line-height: 1;
		min-height: 2em;
	}

	select {
		margin: 0;
	}

	:focus-visible {
		outline: 0.3em solid var(--layer-color, orange);
		outline-offset: 0.2em;
	}

	select:active, select:focus {
		outline: 0.3em solid var(--layer-color, orange);
		outline-offset: 0.2em;
	}

	.big-button {
		width: 100%;
		padding: 2em;
	}
</style>

<div class="container">

<div class="controls">
	<h1>Affine Transformation <br>of Functions</h1>
	
	{#each layers as layer, l}
	<div class="layer-box" style:--layer-color={layerColor(layers, l)}>
		<h3>Layer #{l+1} 
			<span>
				<button on:click={() => reset(l)}>Reset</button>
			<button on:click={() => removeLayer(l)}>Remove</button>
			</span>
		</h3>


		<div class="checkbox-list">
			<div class="checkbox-list">
				<div style:display="flex" style:justify-content="space-between">
					<label for="f_{l}">Function f(x): </label>			<label><input type="checkbox" bind:checked={layer.onlyPositives} /> Only x>=0</label>
				</div>

				<select id="f_{l}" bind:value={layer.selectedFunction}>
					{#each functions as f, i}
						<option value={i}>{f.name}</option>
					{/each}
				</select>
			</div>
			<div class="checkbox-list">
				<strong>Parameters</strong>
				<div class="checkbox-row">
					<label for="param_k_{l}" style:display="inline-block" style:width="8em">K: {(layer.baseParam.integer?intFormat:numFormat).format(layer.baseParam.value)}</label> 
					<input id="param_k_{l}" style:accent-color="var(--layer-color)" type="range" min={0} max="15" step={layer.baseParam.integer?1:0.01} bind:value={layer.baseParam.value} /> 
					<label><input type="checkbox" bind:checked={layer.baseParam.integer} /> Integers only</label>

				</div>
				<div class="checkbox-row">
					<label for="param_a_{l}" style:display="inline-block" style:width="8em">A: {numFormat.format(layer.scale.x)}</label> 
					<label for="param_b_{l}" style:display="inline-block" style:width="8em">B: {numFormat.format(layer.scale.y)}</label> 
					<label for="param_c_{l}" style:display="inline-block" style:width="8em">C: {numFormat.format(layer.offset.x)}</label> 
					<label for="param_d_{l}" style:display="inline-block" style:width="8em">D: {numFormat.format(layer.offset.y)}</label> 

				</div>
			</div>
		</div>
	</div>
	{/each}

	{#if hues.length}
	<button class="big-button" on:click={() => addLayer()}>Add layer</button>
	{/if}

	<label class="zoom-control">
		Zoom
	<input type="range" min={-5} max="5" step={0.05} bind:value={zoomFactor} /> 
	</label>
</div>

<SVGCanvas on:wheel={onWheel} let:minVisible let:maxVisible on:lkdragstart={onMouseDown} on:lkdragend={onMouseUp} on:lkdragmove={onMouseMove}>

		
	<g pointer-events="none">
	{#each layers as layer, l}
	<path d="M{layer.offset.x*zoom} ,{-layer.offset.y*zoom}  h {layer.scale.x*zoom} v {-layer.scale.y*zoom} h {-layer.scale.x*zoom} Z" fill="{layerColor(layers, l)}" fill-opacity="0.05" stroke="{layerColor(layers, l)}" stroke-dasharray="5 5" />
	{/each}
	</g>
	
	<g pointer-events="none">
		<line x1={minVisible.x} y1="0" x2={maxVisible.x} y2="0" stroke="black" vector-effect="non-scaling-stroke"/>
		<line y1={minVisible.y} x1="0" y2={maxVisible.y} x2="0" stroke="black" vector-effect="non-scaling-stroke"/>
		<polygon fill="black" points="{maxVisible.x+5} 0 {maxVisible.x-30} -20 {maxVisible.x-30} 20"/>
		<polygon fill="black" points=" 0 {-maxVisible.y} -20 {-maxVisible.y+30} 20  {-maxVisible.y+30} "/>
		<text y="40" x="{maxVisible.x-30}" font-size="30" text-anchor="middle" dominant-baseline="middle">X</text>
		<text x="-40" y="{minVisible.y+30}" font-size="30" text-anchor="middle" dominant-baseline="middle">Y</text>
	</g>
	<g>
		<path d={fixedSegments(3, 10, zoom, 0, maxVisible.x).slice(1, -1).map((x) => `M${x} -20 V20 `).join("")} stroke="black"/>
		<path d={fixedSegments(3, 10, zoom, 0, minVisible.x).slice(1, -1).map((x) => `M${x} -20 V20 `).join("")} stroke="black"/>
		<path d={fixedSegments(3, 10, zoom, 0, maxVisible.y).slice(1, -1).map((x) => `M-20 ${x} H20 `).join("")} stroke="black"/>
		<path d={fixedSegments(3, 10, zoom, 0, minVisible.y).slice(1, -1).map((x) => `M-20 ${x} H20 `).join("")} stroke="black"/>
		{#each fixedSegments(3, 10, zoom, 0, maxVisible.x).slice(1, -1) as s}
			<text x="{s}" y="40" font-size="20" text-anchor="middle">{numFormat.format(s/zoom)}</text>
		{/each}
		{#each fixedSegments(3, 10, zoom, 0, minVisible.x).slice(1, -1) as s}
			<text x="{s}" y="40" font-size="20" text-anchor="middle">{numFormat.format(s/zoom)}</text>
		{/each}
		{#each fixedSegments(3, 10, zoom, 0, maxVisible.y).slice(1, -1) as s}
			<text y="{s}" x="-40" font-size="20" text-anchor="end" dominant-baseline="middle">{numFormat.format(s/zoom)}</text>
		{/each}
		{#each fixedSegments(3, 10, zoom, 0, minVisible.y).slice(1, -1) as s}
			<text y="{s}" x="-40" font-size="20" text-anchor="end" dominant-baseline="middle">{numFormat.format(s/zoom)}</text>
		{/each}
	</g>
	{#each layers as layer, l (layer.hue)}
		{@const f = functions[layer.selectedFunction]}
		{@const color = layerColor(layers, l)}		
		{@const resLeft = Math.abs(maxVisible.x +layer.offset.x) / 2}
		{@const resRight = Math.abs(-layer.offset.x - minVisible.x) / 2}
		<g pointer-events="none">
			{#if f.fractionalB(layer.baseParam.integer, Math.sign(layer.scale.x))}

			<polyline  stroke-width="3" points={segments(resLeft, layer.offset.x, maxVisible.x/zoom).filter(x => !layer.onlyPositives ||(x/layer.scale.x)/layer.scale.x >= 0).map(x => f.fn((x/layer.scale.x)-layer.offset.x/layer.scale.x, layer.baseParam.castValue, Math.sign(layer.scale.x)) === undefined ? '' : `${x*zoom}, ${clamp(-f.fn(((x-layer.offset.x)/layer.scale.x), layer.baseParam.castValue, Math.sign(layer.scale.x))*layer.scale.y*zoom-layer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y)}`).join(" ")} stroke={color} fill="none"/>	
				{/if}
				{#if f.fractionalB(layer.baseParam.integer, -1*Math.sign(layer.scale.x))}
					<polyline  stroke-width="3" points={segments(resRight, layer.offset.x, minVisible.x/zoom).filter(x => !layer.onlyPositives ||(x/layer.scale.x)/layer.scale.x >= 0).map(x => f.fn((x/layer.scale.x)-layer.offset.x/layer.scale.x, layer.baseParam.castValue, -Math.sign(layer.scale.x)) === undefined ? '' : `${x*zoom}, ${clamp(-f.fn(((x-layer.offset.x)/layer.scale.x), layer.baseParam.castValue, -Math.sign(layer.scale.x))*layer.scale.y*zoom-layer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y)}`).join(" ")} stroke={color} fill="none"/>
				{/if}
		</g>
		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 20} y={-layer.offset.y * zoom - 20} height="40" width="40" fill="none"></rect>
		<rect data-layer={l} data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 20} y={-(layer.offset.y+layer.scale.y) * zoom - 20} height="40" width="40" fill="none" rx="10" ry="10"></rect>

		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 10} y={-layer.offset.y * zoom - 10} height="20" width="20" fill="{layerColor(layers, l)}"></rect>

		<rect data-layer={l} data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y) * zoom - 10} height="20" width="20" fill="{layerColor(layers, l)}" rx="10" ry="10"></rect>
	{/each}
</SVGCanvas>

</div>