<script>
	const numFormat = new Intl.NumberFormat("en-US", {useGrouping:false, minimumFractionDigits: 2, maximumFractionDigits: 2,signDisplay:"auto"});
	const axisFormat = new Intl.NumberFormat("en-US", {useGrouping:false, minimumFractionDigits: 0, maximumFractionDigits: 2,signDisplay:"auto"});
	const numFormatSvg = new Intl.NumberFormat("en-US", {useGrouping:false, minimumFractionDigits: 2, maximumFractionDigits: 2,signDisplay:"auto"});
	const intFormat = new Intl.NumberFormat("en-US", {useGrouping:false, minimumFractionDigits: 0, maximumFractionDigits: 0,signDisplay:"auto"});
	const termFormat = new Intl.NumberFormat("en-US", {useGrouping:false, minimumFractionDigits: 2, maximumFractionDigits: 2,signDisplay:"never"});
	import SVGCanvas from './SVGCanvas.svelte'
	

	
	let zoomFactor = 1
	const zoomBase = 50;

	let colors = ['#ff00aa','#00ccdd','#00aa00']


	let layers = [
	]

	let baseLayer = {
		scale: {x:1,y:1},
		offset: {x:0,y:0},
		baseParam: {
			intger: true,
			value: 2,
			get castValue() { return this.integer ? Math.round(this.value) : this.value }
		},
	}

	let singleLayer = true
	
	
	$: zoom = Math.exp(zoomFactor) * zoomBase

	let press = {
		x:0,
		y:0,
		initialx: 0,
		initialy: 0,
		project: {
			x: 1,
			y: 1,
		}
	}
	let pressOffset = false
	let pressScale = false
	let pressLayer = null		
	let xdominant = true

	let latestLayer = null



	function reset(l) {
		if(l===false) {
			latestLayer = l
			baseLayer.scale.x = 1
			baseLayer.scale.y = 1
			baseLayer.offset.x = 0
			baseLayer.offset.y = 0
			baseLayer.baseParam.integer = false
			baseLayer.baseParam.value = 2
			baseLayer.onlyPositives = false
		} else{
			latestLayer = false
			layers[l].scale.x = 1
			layers[l].scale.y = 1
			layers[l].offset.x = 0
			layers[l].offset.y = 0
			layers[l].baseParam.integer = false
			layers[l].baseParam.value = 2
			layers[l].onlyPositives = false
		}
	}

	function removeLayer(l) {
		colors.push(layers[l].color)
		colors = colors
		layers = [...layers.slice(0, l), ...layers.slice(l+1)]
		latestLayer = null
	}

	function removeAll(l) {
		colors = [colors, ...layers.map(l=>l.color)]
		layers = []
		latestLayer = null
	}

	function addLayer() {
		const color = colors.pop()
		
		colors = colors

		if(color) {
			layers = [{
				color: color,
				scale: {x:1,y:1},
	 			offset: {x:0,y:0},
	 			baseParam: {
	 				intger: true,
	 				value: 2,
	 				get castValue() { return this.integer ? Math.round(this.value) : this.value }
	 			},
	 			onlyPositives: false,
	 			selectedFunction: 2,
			}, ...layers]

			latestLayer = layers.length - 1
		}
	}

	addLayer()
	
	function onMouseDown({detail: local}) {
		if(!local.target.getAttribute('data-control')) {
			return
		}

		press.x = local.x
		press.y = local.y
		press.project.x = local.target.getAttribute('data-project-x') || 1
		press.project.y = local.target.getAttribute('data-project-y') || 1

		const control = local.target.getAttribute('data-control')
		pressLayer = local.target.hasAttribute('data-layer')  ? parseInt(local.target.getAttribute('data-layer'), 10) : false
		pressOffset = control === 'offset'
		pressScale = control === 'scale'
		xdominant = true
		latestLayer = pressLayer

		if(pressOffset) {
			press.initialx = (pressLayer === false ? baseLayer : layers[pressLayer]).offset.x
			press.initialy = (pressLayer === false ? baseLayer : layers[pressLayer]).offset.y
		} else if(pressScale) {
			press.initialx = (pressLayer === false ? baseLayer : layers[pressLayer]).scale.x
			press.initialy = (pressLayer === false ? baseLayer : layers[pressLayer]).scale.y
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

		let px =  press.project.x*dx
		let py =  press.project.y*dy

		const currentLayer = (pressLayer === false ? baseLayer : layers[pressLayer])

		if(pressOffset) {
			currentLayer.offset.x = press.initialx + px/zoom
			currentLayer.offset.y = press.initialy - py/zoom
		} else if(pressScale) {
			currentLayer.scale.x = press.initialx + px/zoom
			currentLayer.scale.y = press.initialy - py/zoom
		}

		layers = layers
		baseLayer = baseLayer
	}

	function segments(resolution, min, max) {
		return Array(Math.round(Math.abs(resolution))).fill(null).map((_,i,all) => min + (i/all.length)*(max-min))
	}
	function fixedSegments(mincount, maxcount, resolution, min, max) {
		const count = Math.ceil(Math.abs((max-min)/resolution))
		const factor = Math.min(Math.max(mincount, count), maxcount)/Math.max(1, count)
		return Array(Math.round(count*factor)).fill(null).map((_,i,all) => min/factor + Math.sign(max-min)*i*resolution/factor)
	}


	
	
	let functions = [
		{previewColor: 'hsl(180, 0%,30%)', supportsScale:(s) => true, shortName: 'constant', unicodeExpression: '1', fractionalB: (bIsInt, x) => true, fn:(x, b) => 1},
		{previewColor: 'hsl(0, 30%,50%)', supportsScale:(s) => true, shortName: 'linear', unicodeExpression: 'x', fractionalB: (bIsInt, x) => true, fn:(x, b) => x},
		{previewColor: 'hsl(100, 80%,40%)', supportsScale:(s) => true, shortName: 'power', unicodeExpression: 'xᵏ', fractionalB: (bIsInt, x) => bIsInt || x>=0, fn:(x, b) => Math.pow(x, b)},
		{previewColor: 'hsl(180, 90%,40%)', supportsScale:(s) => true, shortName: 'root', unicodeExpression: 'ᵏ√x', fractionalB: (bIsInt, x) => bIsInt || x>=0, fn:(x, b) => Math.pow(x, 1/b)},
		{previewColor: 'hsl(240, 90%,60%)', supportsScale:(s) => true, shortName: 'reciprocal', unicodeExpression: '1/xᵏ', fractionalB: (bIsInt, x) => bIsInt || x>0, fn:(x, b, sign) => x=== 0 ? (sign<0?(Infinity*Math.pow(-1,Math.ceil(b))):Infinity) : 1/Math.pow(x,b)},
		{previewColor: 'hsl(290, 100%,45%)', supportsScale:(s) => true, shortName: 'exponential', unicodeExpression: 'kˣ', fractionalB: (bIsInt, x) => true, fn:(x, b) => Math.pow(b, x)},
		{previewColor: 'hsl(300, 100%,50%)', supportsScale:(s) => true, shortName: 'logarithmic', unicodeExpression: 'logₖ(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => x >0 && b>0 ? Math.log(x)/Math.log(b) : (x<0?undefined:(b<1?1:-1)*Infinity)},
		{previewColor: 'hsl(330, 100%,50%)', supportsScale:(s) => true, shortName: 'log-linear', unicodeExpression: 'x⋅logₖ(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => x >0 && b>0 ? x*Math.log(x)/Math.log(b) : (x<0?undefined:0)},
		{previewColor: false, supportsScale:(s) => true, shortName: 'sinusoidal', unicodeExpression: 'sin(x)', fractionalB: (bIsInt, x) => true, fn:(x, b) => Math.sin(x)},
	];

	function clamp(v, min, max) {
		return Math.max(min, Math.min(max, v))
	}

	function onWheel(evt) {
		zoomFactor = clamp(zoomFactor + (-(evt.deltaY)/120/3), -5, 5)
	}

	function axisInterval(range) {
		var x = Math.pow(10.0, Math.floor(Math.log10(range)));
	    if (range / x >= 5)
	        return x;
	    else if (range / (x / 2.0) >= 5)
	        return x / 2.0;
	    else
	        return x / 5.0;
	}

</script>

<style>
	:global(body, html) {
		margin: 0;
		padding: 0;
	}

	h1 {
		font-size: 1.4em;
	}

	.controls {
		background: #fff;
		padding: 1em;
		width: 100%;
		overflow: auto;
		scrollbar-width: none;
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
		margin-bottom: 1em;
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
		user-select: none;
		font-size: small;
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
		justify-content: stretch;
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
		padding: 1em 1em;
	}

	.function-list {
		list-style: none;
		margin: 0.5em 0;
		padding: 0;
		display: flex;
		flex-direction: column;
		gap: 0.5em;
		font-size: 1.2em;
	}

	details {
		margin-top: 3em;
	}

	summary {
		cursor: pointer;
	}

	.tabs {
		display: flex;
		gap: 0.5em;
		border-bottom: 2px solid black;
		margin: 0 0 1em;
	}

	.tab {
		background: none;
		color: #000;
		padding: 0.8em 0.5em 0.2em;
		border: none;
		flex-grow: 1;
	}

	.tab[disabled] {
		background: black;
		color: #fff;
		margin: 0;
		padding: 0.5em;
	}

	.scaling {
		display: grid;
		grid-template-columns: 1fr 1fr;
	}

	.cursor-col-resize {
		cursor: col-resize;
		opacity: 0.2;
	}
	.cursor-ew-resize {
		cursor: ew-resize;
		opacity: 0.2;
	}
	.cursor-nesw-resize {
		cursor: nesw-resize;
	}
	.cursor-ns-resize {
		cursor: ns-resize;
		opacity: 0.2;
	}
	.cursor-nwse-resize {
		cursor: nwse-resize;
	}
	.cursor-row-resize {
		cursor: row-resize;
		opacity: 0.2;
	}
</style>

<div class="container">

<div class="controls">
	<h1>Affine Transformation <br>of Functions</h1>
	


	<div class="tabs">
		<button disabled={singleLayer} class="tab" on:click={()=>singleLayer = true}>
			All functions
		</button>
		<button disabled={!singleLayer} class="tab" on:click={()=>singleLayer = false}>
			Selected functions
		</button>
	</div>

	{#if singleLayer}
	<div class="layer-box" style:--layer-color="#111">

		<h3><span style:flex-grow="1">All Functions</span>
			<span>
			<button on:click={() => reset(false)}>Reset</button>
			</span>
		</h3>


		<ul class="function-list">
			{#each functions as f}
			{#if f.previewColor}	
			<li style:color={f.previewColor}><strong>{f.shortName}:</strong> {f.unicodeExpression}</li>
			{/if}
			{/each}
		</ul>

		<strong>Parameters</strong>
		<div class="checkbox-row">
			<label for="param_k_base" style:display="inline-block" style:width="8em">k: {(baseLayer.baseParam.integer?intFormat:numFormat).format(baseLayer.baseParam.value)}</label> 
			<input id="param_k_base" style:accent-color="var(--baseLayer-color)" type="range" min={0} max="15" step={baseLayer.baseParam.integer?1:0.01} bind:value={baseLayer.baseParam.value} /> 

		</div>

		<strong>Affine Transformation</strong>
		<div class="scaling">
			<label for="param_a_base" style:display="inline-block" style:width="8em">scale x: {numFormat.format(baseLayer.scale.x)}</label> 
			<label for="param_c_base" style:display="inline-block" style:width="8em">shift x: {numFormat.format(baseLayer.offset.x)}</label> 
			<label for="param_b_base" style:display="inline-block" style:width="8em">scale y: {numFormat.format(baseLayer.scale.y)}</label> 
			<label for="param_d_base" style:display="inline-block" style:width="8em">shift y: {numFormat.format(baseLayer.offset.y)}</label> 
		</div>		
		<div style="margin: 1em; text-align: center;font-weight: bold;">
			y = {termFormat.format(baseLayer.scale.y)} ⋅ f({termFormat.format(baseLayer.scale.x)} ⋅ x {baseLayer.offset.x>0?'-':'+'} {termFormat.format(baseLayer.offset.x)}) + {termFormat.format(baseLayer.offset.y)}
		</div>

	</div>

	{:else}

	<div style:display="flex" style:gap="0.5em">
		<button disabled={!colors.length} class="big-button" on:click={() => addLayer()}>
			Add function
		</button>
	</div>

	{#each layers as layer, l (l)}



	<div on:mousedown={() => {latestLayer = l}} class="layer-box" style:--layer-color={layer.color}>
		<h3><label style:flex-grow="1">Function #{layers.length - l} 
			<input type="color" style:display="none" bind:value={layer.color} /></label>
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
						<option value={i}>{f.shortName} {f.unicodeExpression}</option>
					{/each}
				</select>
			</div>
			<div class="checkbox-list">
				<strong>Parameters</strong>
				<div class="checkbox-row">
					<label for="param_k_{l}" style:display="inline-block" style:width="8em">k: {(layer.baseParam.integer?intFormat:numFormat).format(layer.baseParam.value)}</label> 
					<input id="param_k_{l}" style:accent-color="var(--layer-color)" type="range" min={0} max="15" step={layer.baseParam.integer?1:0.01} bind:value={layer.baseParam.value} /> 
					<label><input type="checkbox" bind:checked={layer.baseParam.integer} /> Integers only</label>

				</div>

				<strong>Affine Transformation</strong>

				<div class="scaling">
					<label for="param_a_{l}" style:display="inline-block" style:width="8em">scale x: {numFormat.format(layer.scale.x)}</label> 
					<label for="param_c_{l}" style:display="inline-block" style:width="8em">shift x: {numFormat.format(layer.offset.x)}</label> 
					<label for="param_b_{l}" style:display="inline-block" style:width="8em">scale y: {numFormat.format(layer.scale.y)}</label> 
					<label for="param_d_{l}" style:display="inline-block" style:width="8em">shift y: {numFormat.format(layer.offset.y)}</label> 

				</div>
			</div>
		</div>
	</div>
	{/each}
	{/if}

	<details>
		<summary>Explanation</summary>
		
		<p>
			On the right side you can see how the plots of various functions look in relation on to each other. Use the circular or the square handle in the coordinate system to scale or shift the functions in X or in Y direction. This shifting and scaling is called affine transforming.
		</p>
		<p>
			You will notice that the relative distances between the functions will not change. The parameter <strong>k</strong> can be adjusted using the slider on the left side. You can observe how the curvature of the functions will be affected.
		</p>
		<p>
			You can only click on the <em>pick function</em> button on the left side to select individual functions to compare to each other. Individual functions can be scaled independently from each other. But when comparing two functions you will observe that, depending on the function and the parameter <strong>k</strong>, one will grow much higher and faster no matter how the other function is scaled. Use the zoom slider (or mousewheel) in the bottom right corner to zoom out and observe the behavior for the functions for larger <strong>x</strong> values. 
		</p>
		<p>
			<strong>In short: Affine transformations do not affect a functions general behavior for large input values.</strong>
		</p>
	</details>

	<p style:text-align="center">
		<a href="https://tools.laszlokorte.de" title="More educational tools">More educational tools</a>
	</p>

	<label class="zoom-control" on:wheel|preventDefault={onWheel} on:dblclick|preventDefault={() => zoomFactor = 1}>
		Zoom <input type="range" min={-5} max="5" step={0.05} bind:value={zoomFactor} /> 
	</label>
</div>

<SVGCanvas on:wheel={onWheel} let:minVisible let:maxVisible on:lkdragstart={onMouseDown} on:lkdragend={onMouseUp} on:lkdragmove={onMouseMove}>
	{@const arrowsize = Math.max(Math.abs(maxVisible.y - minVisible.y), Math.abs(maxVisible.x - minVisible.x)) / 80}

	{@const maxSize = Math.max(maxVisible.x, maxVisible.y)}
	{@const maxAxis = (maxSize / zoom)}
	{@const tickWidth = axisInterval(maxAxis)}
	{@const xTickCount = Math.floor(maxVisible.x / zoom / tickWidth)}
	{@const yTickCount = Math.floor(maxVisible.x / zoom / tickWidth)}
	{@const xTickLength = xTickCount * tickWidth * zoom}
	{@const yTickLength = yTickCount * tickWidth * zoom}

		
	<g pointer-events="none">
	{#if !singleLayer}
	{#each layers as layer, l (l)}
	{@const maxw = Math.abs(maxVisible.x - minVisible.x)}
	{@const maxh = Math.abs(maxVisible.y - minVisible.y)}

	<path d="M{numFormatSvg.format(clamp(layer.offset.x*zoom, -maxw, maxw))},{numFormatSvg.format(clamp(-layer.offset.y*zoom, -maxh, maxh))}  H {numFormatSvg.format(clamp((layer.scale.x+layer.offset.x)*zoom, -maxw, maxw))} V {numFormatSvg.format(clamp(-(layer.scale.y+layer.offset.y)*zoom, -maxh, maxh))} H {numFormatSvg.format(clamp(layer.offset.x*zoom, -maxw, maxw))} Z" fill="{layer.color}" fill-opacity="0.05" stroke="{layer.color}" stroke-dasharray="5 5" />
	{/each}
	{:else}
	{@const maxw = Math.abs(maxVisible.x - minVisible.x)}
	{@const maxh = Math.abs(maxVisible.y - minVisible.y)}

	<path d="M{numFormatSvg.format(clamp(baseLayer.offset.x*zoom, -maxw, maxw))},{numFormatSvg.format(clamp(-baseLayer.offset.y*zoom, -maxh, maxh))}  H {numFormatSvg.format(clamp((baseLayer.scale.x+baseLayer.offset.x)*zoom, -maxw, maxw))} V {numFormatSvg.format(clamp(-(baseLayer.scale.y+baseLayer.offset.y)*zoom, -maxh, maxh))} H {numFormatSvg.format(clamp(baseLayer.offset.x*zoom, -maxw, maxw))} Z" fill="gray" fill-opacity="0.05" stroke="gray" stroke-dasharray="5 5" />
	{/if}
	</g>
	
	<g pointer-events="none">
		<line x1={minVisible.x} y1="0" x2={maxVisible.x} y2="0" stroke="black" vector-effect="non-scaling-stroke"/>
		<line y1={minVisible.y} x1="0" y2={maxVisible.y} x2="0" stroke="black" vector-effect="non-scaling-stroke"/>
		<polygon fill="black" points="{maxVisible.x+5} 0 {maxVisible.x-arrowsize} {-arrowsize/2} {maxVisible.x-arrowsize} {arrowsize/2}"/>
		<polygon fill="black" points=" 0 {-maxVisible.y} {-arrowsize/2} {-maxVisible.y+arrowsize} {arrowsize/2}  {-maxVisible.y+arrowsize} "/>
		<text y="{arrowsize*2}" x="{maxVisible.x-arrowsize}" font-size={arrowsize} text-anchor="middle" dominant-baseline="middle">X</text>
		<text x="{-arrowsize*2}" y="{minVisible.y+arrowsize}" font-size={arrowsize} text-anchor="middle" dominant-baseline="middle">Y</text>
	</g>
	<g>
		<path vector-effect="non-scaling-stroke" d={segments(xTickCount, 0, xTickLength).slice(1).map((x) => `M${x} ${-arrowsize} V${arrowsize} `).join("")} stroke="black"/>
		<path vector-effect="non-scaling-stroke" d={segments(xTickCount, 0, -xTickLength).slice(1).map((x) => `M${x} ${-arrowsize} V${arrowsize} `).join("")} stroke="black"/>
		<path vector-effect="non-scaling-stroke" d={segments(yTickCount, 0, yTickLength).slice(1).map((x) => `M${-arrowsize} ${x} H${arrowsize} `).join("")} stroke="black"/>
		<path vector-effect="non-scaling-stroke" d={segments(yTickCount, 0, -yTickLength).slice(1).map((x) => `M${-arrowsize} ${x} H${arrowsize} `).join("")} stroke="black"/>
		{#each segments(xTickCount, 0, xTickLength).slice(1) as s, si}
			{#if !si || si%2 == 1}
			<text x="{s}" y="{arrowsize*2}" font-size={arrowsize} text-anchor="middle">{axisFormat.format(s/zoom)}</text>
			{/if}
		{/each}
		{#each segments(xTickCount, 0, -xTickLength).slice(1) as s, si}
			{#if !si || si%2 == 1}
			<text x="{s}" y="{arrowsize*2}" font-size={arrowsize} text-anchor="middle">{axisFormat.format(s/zoom)}</text>
			{/if}
		{/each}
		{#each segments(yTickCount, 0, -yTickLength).slice(1) as s, si}
			{#if !si || si%2 == 1}
			<text y="{s}" x="{-arrowsize*1.5}" font-size={arrowsize} text-anchor="end" dominant-baseline="middle">{axisFormat.format(-s/zoom)}</text>
			{/if}
		{/each}
		{#each segments(yTickCount, 0, yTickLength).slice(1) as s, si}
			{#if !si || si%2 == 1}
			<text y="{s}" x="{-arrowsize*1.5}" font-size={arrowsize} text-anchor="end" dominant-baseline="middle">{axisFormat.format(-s/zoom)}</text>
			{/if}
		{/each}
	</g>
	{#if !singleLayer}
	{#each layers as layer, l (l)}
		{@const f = functions[layer.selectedFunction]}
		{@const resLeft = clamp(maxVisible.x +layer.offset.x*zoom, 0, maxVisible.x-minVisible.x) / 2}
		{@const resRight = clamp(-layer.offset.x*zoom - minVisible.x, 0, maxVisible.x-minVisible.x) / 2}
		<g pointer-events="none">
			{#if f.fractionalB(layer.baseParam.integer, Math.sign(layer.scale.x))}

			<polyline  vector-effect="non-scaling-stroke" stroke-width="3" points={segments(resRight, layer.offset.x, (maxVisible.x + 10)/zoom).filter(x => !layer.onlyPositives ||(x/layer.scale.x)/layer.scale.x >= 0).map(x => f.fn((x/layer.scale.x)-layer.offset.x/layer.scale.x, layer.baseParam.castValue, Math.sign(layer.scale.x)) === undefined ? '' : `${numFormatSvg.format(x*zoom)}, ${numFormatSvg.format(clamp(-f.fn(((x-layer.offset.x)/layer.scale.x), layer.baseParam.castValue, Math.sign(layer.scale.x))*layer.scale.y*zoom-layer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y))}`).join(" ")} stroke={layer.color} fill="none"/>	
				{/if}
				{#if f.fractionalB(layer.baseParam.integer, -1*Math.sign(layer.scale.x))}
					<polyline  vector-effect="non-scaling-stroke" stroke-width="3" points={segments(resLeft, layer.offset.x, (minVisible.x - 10)/zoom).filter(x => !layer.onlyPositives ||(x/layer.scale.x)/layer.scale.x >= 0).map(x => f.fn((x/layer.scale.x)-layer.offset.x/layer.scale.x, layer.baseParam.castValue, -Math.sign(layer.scale.x)) === undefined ? '' : `${numFormatSvg.format(x*zoom)}, ${numFormatSvg.format(clamp(-f.fn(((x-layer.offset.x)/layer.scale.x), layer.baseParam.castValue, -Math.sign(layer.scale.x))*layer.scale.y*zoom-layer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y))}`).join(" ")} stroke={layer.color} fill="none"/>
				{/if}
		</g>
		
		<g>
		<rect data-layer={l} data-project-x=0 class="cursor-row-resize" data-control="offset" x={(layer.scale.x/2+layer.offset.x) * zoom - 25} y={-(layer.offset.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-layer={l} data-project-y=0 class="cursor-col-resize" data-control="offset" x={(layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>

		<rect data-layer={l} data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(layer.scale.x/2+layer.offset.x) * zoom - 25} y={-(layer.offset.y+layer.scale.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-layer={l} data-project-y=0 class="cursor-ew-resize" data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>



		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 20} y={-layer.offset.y * zoom - 20} height="40" width="40" fill="none" stroke="none"></rect>
		<rect data-layer={l} data-control="scale" class="cursor-{layer.scale.x*layer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" x={(layer.scale.x+layer.offset.x) * zoom - 20} y={-(layer.offset.y+layer.scale.y) * zoom - 20} height="40" width="40" fill="none" rx="20" ry="20" stroke="none"></rect>


		<rect  data-layer={l} data-project-x=0 class="cursor-row-resize" data-control="offset" x={(layer.scale.x/2+layer.offset.x) * zoom - 20} y={-(layer.offset.y) * zoom - 5} height="10" width="40" fill="{layer.color}" stroke="white"></rect>
		
		<rect  data-layer={l} data-project-y=0 class="cursor-col-resize" data-control="offset" x={(layer.offset.x) * zoom - 5} y={-(layer.offset.y+layer.scale.y/2) * zoom - 20} height="40" width="10" fill="{layer.color}" stroke="white"></rect>


		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 10} y={-layer.offset.y * zoom - 10} height="20" width="20" fill="{layer.color}" stroke="white"></rect>
		
		<rect  data-layer={l} data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(layer.scale.x/2+layer.offset.x) * zoom - 20} y={-(layer.offset.y+layer.scale.y) * zoom - 5} height="10" width="40" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>
			
		<rect  data-layer={l} data-project-y=0  class="cursor-ew-resize" data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 5} y={-(layer.offset.y+layer.scale.y/2) * zoom - 20} height="40" width="10" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>

		<rect data-layer={l} data-control="scale" class="cursor-{layer.scale.x*layer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" x={(layer.scale.x+layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y) * zoom - 10} height="20" width="20" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>
	</g>

	{/each}
	{#if layers[latestLayer]}
	{@const l = latestLayer} 
	{@const layer = layers[latestLayer]} 

	<g>
		<rect data-layer={l} data-project-x=0 class="cursor-row-resize" data-control="offset" x={(layer.scale.x/2+layer.offset.x) * zoom - 25} y={-(layer.offset.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-layer={l} data-project-y=0 class="cursor-col-resize" data-control="offset" x={(layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>

		<rect data-layer={l} data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(layer.scale.x/2+layer.offset.x) * zoom - 25} y={-(layer.offset.y+layer.scale.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-layer={l} data-project-y=0 class="cursor-ew-resize" data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>



		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 20} y={-layer.offset.y * zoom - 20} height="40" width="40" fill="none" stroke="none"></rect>
		<rect data-layer={l} data-control="scale" class="cursor-{layer.scale.x*layer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" x={(layer.scale.x+layer.offset.x) * zoom - 20} y={-(layer.offset.y+layer.scale.y) * zoom - 20} height="40" width="40" fill="none" rx="20" ry="20" stroke="none"></rect>


		<rect  data-layer={l} data-project-x=0 class="cursor-row-resize" data-control="offset" x={(layer.scale.x/2+layer.offset.x) * zoom - 20} y={-(layer.offset.y) * zoom - 5} height="10" width="40" fill="{layer.color}" stroke="white"></rect>
		
		<rect  data-layer={l} data-project-y=0 class="cursor-col-resize" data-control="offset" x={(layer.offset.x) * zoom - 5} y={-(layer.offset.y+layer.scale.y/2) * zoom - 20} height="40" width="10" fill="{layer.color}" stroke="white"></rect>


		<rect data-layer={l} data-control="offset" x={layer.offset.x * zoom - 10} y={-layer.offset.y * zoom - 10} height="20" width="20" fill="{layer.color}" stroke="white"></rect>
		
		<rect  data-layer={l} data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(layer.scale.x/2+layer.offset.x) * zoom - 20} y={-(layer.offset.y+layer.scale.y) * zoom - 5} height="10" width="40" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>
			
		<rect  data-layer={l} data-project-y=0  class="cursor-ew-resize" data-control="scale" x={(layer.scale.x+layer.offset.x) * zoom - 5} y={-(layer.offset.y+layer.scale.y/2) * zoom - 20} height="40" width="10" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>

		<rect data-layer={l} data-control="scale" class="cursor-{layer.scale.x*layer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" x={(layer.scale.x+layer.offset.x) * zoom - 10} y={-(layer.offset.y+layer.scale.y) * zoom - 10} height="20" width="20" fill="{layer.color}" rx="10" ry="10" stroke="white"></rect>
	</g>

	{/if}
	{:else}
		{@const resLeft = clamp(maxVisible.x +baseLayer.offset.x*zoom, 0, maxVisible.x-minVisible.x) / 2}
		{@const resRight = clamp(-baseLayer.offset.x*zoom - minVisible.x, 0, maxVisible.x-minVisible.x) / 2}
		{@const segmentsLeft = segments(resLeft, baseLayer.offset.x, (minVisible.x - 10)/zoom )}
		{@const segmentsRight = segments(resRight, baseLayer.offset.x, (maxVisible.x + 10)/zoom)}
		{#each functions as f, fi}
		{#if f.previewColor}
		<g pointer-events="none">
			{#if f.fractionalB(baseLayer.baseParam.integer, Math.sign(baseLayer.scale.x))}

			<polyline  vector-effect="non-scaling-stroke" stroke-width="3" points={segmentsRight.filter(x => !baseLayer.onlyPositives ||(x/baseLayer.scale.x)/baseLayer.scale.x >= 0).map(x => f.fn((x/baseLayer.scale.x)-baseLayer.offset.x/baseLayer.scale.x, baseLayer.baseParam.castValue, Math.sign(baseLayer.scale.x)) === undefined ? '' : `${numFormatSvg.format(x*zoom)}, ${numFormatSvg.format(clamp(-f.fn(((x-baseLayer.offset.x)/baseLayer.scale.x), baseLayer.baseParam.castValue, Math.sign(baseLayer.scale.x))*baseLayer.scale.y*zoom-baseLayer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y))}`).join(" ")} stroke={f.previewColor} fill="none"/>	
				{/if}
				{#if f.fractionalB(baseLayer.baseParam.integer, -1*Math.sign(baseLayer.scale.x))}
					<polyline  vector-effect="non-scaling-stroke" stroke-width="3" points={segmentsLeft.filter(x => !baseLayer.onlyPositives ||(x/baseLayer.scale.x)/baseLayer.scale.x >= 0).map(x => f.fn((x/baseLayer.scale.x)-baseLayer.offset.x/baseLayer.scale.x, baseLayer.baseParam.castValue, -Math.sign(baseLayer.scale.x)) === undefined ? '' : `${numFormatSvg.format(x*zoom)}, ${numFormatSvg.format(clamp(-f.fn(((x-baseLayer.offset.x)/baseLayer.scale.x), baseLayer.baseParam.castValue, -Math.sign(baseLayer.scale.x))*baseLayer.scale.y*zoom-baseLayer.offset.y*zoom, 3*minVisible.y, zoom*3*maxVisible.y))}`).join(" ")} stroke={f.previewColor} fill="none"/>
				{/if}
		</g>
		{/if}
		{/each}


		<rect data-project-x=0 class="cursor-row-resize" data-control="offset" x={(baseLayer.scale.x/2+baseLayer.offset.x) * zoom - 25} y={-(baseLayer.offset.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-project-y=0 class="cursor-col-resize" data-control="offset" x={(baseLayer.offset.x) * zoom - 10} y={-(baseLayer.offset.y+baseLayer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>

		<rect data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(baseLayer.scale.x/2+baseLayer.offset.x) * zoom - 25} y={-(baseLayer.offset.y+baseLayer.scale.y) * zoom - 10} height="20" width="50" fill="none" stroke="none"></rect>
		
		<rect data-project-y=0 class="cursor-ew-resize" data-control="scale" x={(baseLayer.scale.x+baseLayer.offset.x) * zoom - 10} y={-(baseLayer.offset.y+baseLayer.scale.y/2) * zoom - 25} height="50" width="20" fill="none" stroke="none"></rect>


		<rect data-control="offset" x={baseLayer.offset.x * zoom - 20} y={-baseLayer.offset.y * zoom - 20} height="40" width="40" fill="none"></rect>

		<rect class="cursor-{baseLayer.scale.x*baseLayer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" data-control="scale" x={(baseLayer.scale.x+baseLayer.offset.x) * zoom - 20} y={-(baseLayer.offset.y+baseLayer.scale.y) * zoom - 20} height="40" width="40" fill="none" rx="10" ry="10"></rect>


		<rect data-project-x=0 class="cursor-row-resize" data-control="offset" x={(baseLayer.scale.x/2+baseLayer.offset.x) * zoom - 20} y={-(baseLayer.offset.y) * zoom - 5} height="10" width="40" fill="#27f" stroke="white"></rect>
		
		<rect data-project-y=0 class="cursor-col-resize" data-control="offset" x={(baseLayer.offset.x) * zoom - 5} y={-(baseLayer.offset.y+baseLayer.scale.y/2) * zoom - 20} height="40" width="10" fill="#27f" stroke="white"></rect>


		<rect data-control="offset" x={baseLayer.offset.x * zoom - 10} y={-baseLayer.offset.y * zoom - 10} height="20" width="20" fill="#27f" stroke="white"></rect>

		<rect data-project-x=0 class="cursor-ns-resize" data-control="scale" x={(baseLayer.scale.x/2+baseLayer.offset.x) * zoom - 20} y={-(baseLayer.offset.y+baseLayer.scale.y) * zoom - 5} height="10" width="40" fill="#27f" rx="10" ry="10" stroke="white"></rect>
		
		<rect data-project-y=0  class="cursor-ew-resize" data-control="scale" x={(baseLayer.scale.x+baseLayer.offset.x) * zoom - 5} y={-(baseLayer.offset.y+baseLayer.scale.y/2) * zoom - 20} height="40" width="10" fill="#27f" rx="10" ry="10" stroke="white"></rect>

		<rect class="cursor-{baseLayer.scale.x*baseLayer.scale.y > 0 ? 'nesw-resize' : 'nwse-resize'}" data-control="scale" x={(baseLayer.scale.x+baseLayer.offset.x) * zoom - 10} y={-(baseLayer.offset.y+baseLayer.scale.y) * zoom - 10} height="20" width="20" fill="#27f" rx="10" ry="10" stroke="white"></rect>
		
	{/if}
</SVGCanvas>

</div>