<script>	
	import { createEventDispatcher, onMount } from 'svelte';
	const dispatch = createEventDispatcher();
	
	let viewBox = {
		minX: -500,
		minY: -500,
		width: 1000,
		height: 1000,
		alignmentX: 'Mid',
		alignmentY: 'Mid',
		scaling: 'meet',
	}

	let element =  {
		width: 400,
		height: 400,
	}

	function scaleViewBox(viewBox, elementWidth, elementHeight) {
		if(viewBox.scaling === 'none') {
			return {
				minX: viewBox.minX,
				minY: viewBox.minY,
				width: viewBox.width,
				height: viewBox.height,
			}
		} else {
			const relWidth = viewBox.width/elementWidth
			const relHeight = viewBox.height/elementHeight
			
			const factor = {
				'meet': Math.max,
				'slice': Math.min
			}[viewBox.scaling].call(Math, relWidth, relHeight)

			const actualWidth = elementWidth * factor
			const actualHeight = elementHeight * factor
			const extraWidth = actualWidth - viewBox.width
			const extraHeight = actualHeight - viewBox.height

			const alignmentWeights = {
				'Min': 0,
				'Mid': 0.5,
				'Max': 1,
			};
			
			const extraWeightingX = alignmentWeights[viewBox.alignmentX];
			const extraWeightingY = alignmentWeights[viewBox.alignmentY];

			return {
				minX:  viewBox.minX - extraWeightingX * extraWidth,
				minY: viewBox.minY - extraWeightingY * extraHeight,
				width: actualWidth,
				height: actualHeight,
			}
		}
	}

	function screenToSVG(x,y,elementX, elementY, elementWidth, elementHeight, localWidth, localHeight, viewBox) {
			const offsetX = x - elementX
			const offsetY = y - elementY
			const relativeX = offsetX / elementWidth
			const relativeY = offsetY / elementHeight

			const scaledVB = scaleViewBox(viewBox, localWidth, localHeight)

			return {
				x: scaledVB.minX + scaledVB.width * relativeX,
				y: scaledVB.minY + scaledVB.height * relativeY,
			}
	}

	function eventToPos(evt, target) {
		const targetRect = target.getBoundingClientRect();
		
		return screenToSVG(
			evt.clientX, evt.clientY,
			targetRect.left, targetRect.top, 
			targetRect.width, targetRect.height, 
			target.clientWidth, target.clientHeight, 
			viewBox
		)
	}

	let initialShift = false
	let initialCtrl = false
	let initialAlt = false
	let leftMouse = false
	let rightMouse = false
	let mouseDownTarget = null
	
	function onMouseDown(evt) {
		evt.preventDefault()
		initialShift = evt.shiftKey
		initialCtrl = evt.ctrlKey
		initialAlt = evt.altKey
		if(evt.button === 0)
			leftMouse = true
		if(evt.button === 2)
			rightMouse = true
		mouseDownTarget = evt.currentTarget
		dispatch('lkdragstart', {...eventToPos(evt, mouseDownTarget), initialShift, initialAlt, initialCtrl, shift: initialShift, alt: initialAlt, ctrl: initialCtrl, leftMouse, rightMouse, target:evt.target})
		onMouseMove(evt)
	}

	function onMouseUp(evt) {
		if(!mouseDownTarget) {
			return
		}
		if(evt.button === 0)
			leftMouse = false
		if(evt.button === 2)
			rightMouse = false
		dispatch('lkdragend', eventToPos(evt, mouseDownTarget))
		if(!leftMouse && !rightMouse)
			mouseDownTarget = null
	}

	function onMouseMove(evt) {
		if(!mouseDownTarget) {
			return
		}
		dispatch('lkdragmove', {...eventToPos(evt, mouseDownTarget), shift: evt.shiftKey, ctrl: evt.ctrlKey, alt: evt.altKey, leftMouse, rightMouse, initialShift, initialAlt, initialCtrl})
	}

	function aspectRatioString(viewBox) {
		if(viewBox.scaling === 'none') {
			return 'none'
		} else {
			return `x${viewBox.alignmentX}Y${viewBox.alignmentY} ${viewBox.scaling}`
		}
	}


	function viewBoxString(viewBox) {
		return `${viewBox.minX} ${viewBox.minY} ${viewBox.width} ${viewBox.height}`
	}

	let innerWidth = 1
	let innerHeight = 1;

	$: minVisible = screenToSVG(
			0, 0,
			0, 0, 
			innerWidth, innerHeight, 
			innerWidth, innerHeight, 
			viewBox
		)

	$: maxVisible = screenToSVG(
			innerWidth, innerHeight, 
			0, 0, 
			innerWidth, innerHeight, 
			innerWidth, innerHeight, 
			viewBox
		)

	let svg

	onMount(() => {
		innerWidth = svg.clientWidth
		innerHeight = svg.clientHeight

		function resize() {
			innerWidth = svg.clientWidth
			innerHeight = svg.clientHeight
		}

		window.addEventListener('resize', resize)

		return () => {
			window.removeEventListener('resize', resize)
		}
	})
</script>

<style>
	svg {
		display: block;
		width: 100%;
		height: 100%;
	}
</style>

<svelte:window on:mousemove={onMouseMove} on:mouseup={onMouseUp} />


<svg bind:this={svg} on:wheel role="button" tabindex="0" on:keydown|preventDefault on:mousedown={onMouseDown} width={element.width} height={element.height} viewBox={viewBoxString(viewBox)} preserveAspectRatio={aspectRatioString(viewBox)}>
	<slot {maxVisible} {minVisible}></slot>	
</svg>