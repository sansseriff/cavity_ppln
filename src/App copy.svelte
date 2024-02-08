<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import * as math from 'mathjs';

	export let name;

	let canvas; // reference to the canvas element

	onMount(() => {
		const random_array = math.reshape(d3.shuffle(d3.range(100)), [10, 10])
		console.log(random_array)
		imshow(random_array, 20, d3.scaleSequential(d3.interpolateViridis));
	});

	function imshow(data, pixelSize, color) {
		// Flatten 2D input array
		const flat = [].concat.apply([], data);
		// Color Scale & Min-Max normalization
		const [min, max] = d3.extent(flat);
		const normalize = (d) => (d - min) / (max - min);
		const colorScale = (d) => color(normalize(d));
		// Shape of input array
		const shape = { x: data[0].length, y: data.length };

		// Set up canvas element
		const context = canvas.getContext("2d");
		canvas.width = shape.x * pixelSize;
		canvas.height = shape.y * pixelSize;
		canvas.style.width = `${shape.x * pixelSize}px`;
		canvas.style.height = `${shape.y * pixelSize}px`;
		canvas.style.imageRendering = "pixelated";

		// Draw pixels to the canvas
		const imageData = context.createImageData(shape.x, shape.y);
		flat.forEach((d, i) => {
			let color = isNaN(d)
				? { r: 0, g: 0, b: 0 }
				: d3.color(colorScale(d));
			imageData.data[i * 4] = color.r;
			imageData.data[i * 4 + 1] = color.g;
			imageData.data[i * 4 + 2] = color.b;
			imageData.data[i * 4 + 3] = 255;
		});
		context.putImageData(imageData, 0, 0);
	}
</script>

<main>
	<h1>Hello {name}!</h1>
	<p>
		Visit the <a href="https://svelte.dev/tutorial">Svelte tutorial</a> to learn
		how to build Svelte apps.
	</p>
	<canvas bind:this={canvas}></canvas> <!-- bind the canvas element -->
</main>

<style>
	/* your styles here */
</style>