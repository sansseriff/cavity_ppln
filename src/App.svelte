<script>
	import { onMount, afterUpdate } from "svelte";
	import * as d3 from "d3";
	import * as math from "mathjs";

	import {
		linspace,
		Sellimeier_PPLN,
		AiryFunction,
		finesse,
		WL_by_energy_conservation,
	} from "./util.js";

	// export let name;

	let canvas; // reference to the canvas element
	let svg; // reference to the svg element
	let cvs_width;
	let cvs_height;
	let isMounted = false;

	const params = {
		c: 299792458, // speed of light in m/s
		WLpumpair: 780e-9,
		numWLs: 501,
		BW: 10e-9, // bandwidth in m
		WLSigma: 10, // sigma in nm
		L: 0.0074, // length in mm
		R1: 0.85,
		R2: 0.1,
		Texpt: 128.25
	};

	const WLCenteral = params.WLpumpair * 2;

	function compute_result(p) {
		const Omega0 = (2 * Math.PI * p.c) / (2 * p.WLpumpair);
		const WLCenteral = p.WLpumpair * 2;

		const WLSigma = p.WLSigma*1e-9;
		const L = p.L*1e-3;

		const WLSignalairJSI = linspace(
			WLCenteral - p.BW / 2,
			WLCenteral + p.BW / 2,
			p.numWLs,
		);
		const WLIdlerairJSI = linspace(
			WLCenteral - p.BW / 2,
			WLCenteral + p.BW / 2,
			p.numWLs,
		);


		// const nPumpPPLN = Sellimeier_PPLN(p.WLpumpair, "eray", p.Texpt)
		const nSignalPPLN= Sellimeier_PPLN(WLSignalairJSI, "eray", p.Texpt)
    	const nIdlerPPLN= Sellimeier_PPLN(WLIdlerairJSI, "eray", p.Texpt)

		// console.log("nSignalPPLN: ", nSignalPPLN)

		const ASignal = AiryFunction(p.R1, p.R2, p.L , nSignalPPLN, WLSignalairJSI)
    	const AIdler = AiryFunction(p.R1, p.R2, p.L, nIdlerPPLN, WLIdlerairJSI)



		const OmegaSignalJSI = WLSignalairJSI.map(
			(wl) => (2 * Math.PI * p.c) / wl,
		);
		let OmegaIdlerJSI = WLIdlerairJSI.map((wl) => (2 * Math.PI * p.c) / wl);
		const OmegaSignalGrid = math.matrix(OmegaSignalJSI);
		const OmegaIdlerGrid = math.matrix(OmegaIdlerJSI);

		// const p.
		const FSigma = (WLSigma * p.c) / (2 * p.WLpumpair) ** 2;

		// Calculate the JSI using the Gaussian approximation

		// console.log(AIdler)
		// flip the idler array
		OmegaIdlerJSI = OmegaIdlerJSI.reverse();
		let JSI = OmegaSignalJSI.map((os, i) => {
			return OmegaIdlerJSI.map((oi, j) => {
				// if (i === 0 && j === 0) {
				// 	console.log("wl_os: ", (2 * Math.PI * p.c) / os);
				// 	console.log("wl_oi: ", (2 * Math.PI * p.c) / oi);
				// 	// console.log("os: ", os);
				// 	// console.log("oi: ", oi);
				// 	// console.log("Omega0: ", Omega0);
				// 	// console.log("FSigma: ", FSigma);
				// }
				// if (i === p.numWLs - 1 && j === p.numWLs - 1) {
				// 	console.log("wl_os: ", (2 * Math.PI * p.c) / os);
				// 	console.log("wl_oi: ", (2 * Math.PI * p.c) / oi);
				// 	// console.log("os: ", os);
				// 	// console.log("oi: ", oi);
				// 	// console.log("Omega0: ", Omega0);
				// 	// console.log("FSigma: ", FSigma);
				// }
				return AIdler[j] * ASignal[i] 
				// *
				// Math.exp(
				// 	-Math.pow(os + oi - 2 * Omega0, 2) /
				// 		(2 * Math.pow(FSigma, 2)),
				// );
			});
		});

		imshow(JSI, 1, d3.scaleSequential(d3.interpolateViridis));
	}

	$: if (isMounted) {
		// console.log("firing");
		compute_result(params);
	}


	let wl_start = Math.round((WLCenteral - params.BW / 2) * 1e9);
	console.log("wl_start: ", wl_start);

	let wl_end = Math.round((WLCenteral + params.BW / 2) * 1e9);
	console.log("wl_end: ", wl_end);

	console.log(d3.range(100));

	onMount(() => {
		// const random_array = math.reshape(
		// 	d3.range(100).map(d=>d*3),
		// 	[10, 10],
		// );
		// console.log(random_array);
		compute_result(params);

		// Create axes
		const xAxis = d3.axisBottom(
			d3.scaleLinear().domain([wl_start, wl_end]).range([0, cvs_width]),
		);
		const yAxis = d3.axisLeft(
			d3.scaleLinear().domain([wl_start, wl_end]).range([cvs_height, 0]),
		);

		// Append axes to SVG
		d3.select(svg)
			.append("g")
			.attr("transform", `translate(100,${100 + cvs_height})`)
			.call(xAxis);

		d3.select(svg)
			.append("g")
			.attr("transform", `translate(100,100)`)
			.call(yAxis);

		// Set the width and height of the SVG
		svg.setAttribute("width", `${cvs_width + 200}px`);
		svg.setAttribute("height", `${cvs_height + 200}px`);
		isMounted = true;
	});

	// afterUpdate(() => {
		
		
	// });

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
		canvas.width = shape.x;
		canvas.height = shape.y;
		cvs_width = shape.x * pixelSize;
		cvs_height = shape.y * pixelSize;
		canvas.style.width = `${shape.x * pixelSize}px`;
		canvas.style.height = `${shape.y * pixelSize}px`;
		canvas.style.top = `${100}px`;
		canvas.style.left = `${100}px`;
		canvas.style.opacity = 1;
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
			// if (i === 0) {
			// 	console.log("start d: ", d);
			// 	console.log("color: ", color);
			// }
			// if (i === flat.length - 1) {
			// 	console.log("end d: ", d);
			// 	console.log("color: ", color);
			// }
		});
		context.putImageData(imageData, 0, 0);
	}
</script>

<main>
	<canvas bind:this={canvas} style="position: absolute;"></canvas>
	<svg bind:this={svg} style="position: absolute; left: 0; top: 0;"></svg>

	<div class="slidecontainer" style="position: relative; top: 650px;">
		<p class="output">Wavelength Sigma</p>
        <input
            type="range"
            min="0.20"
            max="10"
			step="0.01"
            bind:value={params.WLSigma}
            class="slider"
            id="myRange_1"
        />
		<p class="output">{params.WLSigma} nm</p>
    </div>
	<div class="slidecontainer" style="position: relative; top: 650px;">
		<p class="output">Waveguide Length</p>
        <input
            type="range"
            min="0.0001"
            max="0.002"
			step="0.00001"
            bind:value={params.L}
            class="slider"
            id="myRange_1"
        />
		<p class="output">{params.L} nm</p>
    </div>
	<div class="slidecontainer" style="position: relative; top: 650px;">
		<p class="output">Temperature</p>
        <input
            type="range"
            min="0"
            max="200"
			step="1"
            bind:value={params.Texpt}
            class="slider"
            id="myRange_1"
        />
		<p class="output">{params.Texpt} Celsius</p>
    </div>
	<div class="slidecontainer" style="position: relative; top: 650px;">
		<p class="output">R1</p>
        <input
            type="range"
            min="0"
            max="1"
			step=".0001"
            bind:value={params.R1}
            class="slider"
            id="myRange_1"
        />
		<p class="output">{params.R1}</p>
    </div>
	<div class="slidecontainer" style="position: relative; top: 650px;">
		<p class="output">R2</p>
        <input
            type="range"
            min="0"
            max="1"
			step=".0001"
            bind:value={params.R2}
            class="slider"
            id="myRange_1"
        />
		<p class="output">{params.R2}</p>
    </div>
    
</main>

<style>
	.slidecontainer {
        width: 700px; /* Width of the outside container */
        /* padding-top: 1rem; */
		/* background-color: #04aa6d; */
		display: flex;
		flex-direction: row;
		margin-left: 10px;
    }

	/* The slider itself */
    .slider {
        -webkit-appearance: none; /* Override default CSS styles */
        appearance: none;
        width: 50%; /* Full-width */
        height: 25px; 
        background: #e5e5e5; /* Grey background */
        outline: none; /* Remove outline */
        opacity: 0.7; /* Set transparency (for mouse-over effects on hover) */
        -webkit-transition: 0.2s; /* 0.2 seconds transition on hover */
        transition: opacity 0.2s;
        border-radius: 0.3rem;
		margin-top: 13px;
    }

	/* Mouse-over effects */
    .slider:hover {
        opacity: 1; /* Fully shown on mouse-over */
    }

    /* The slider handle (use -webkit- (Chrome, Opera, Safari, Edge) and -moz- (Firefox) to override default look) */
    .slider::-webkit-slider-thumb {
        -webkit-appearance: none; /* Override default look */
        appearance: none;
        width: 25px; /* Set a specific slider handle width */
        height: 25px; /* Slider handle height */
        background: #9a9a9a; /* Green background */
        cursor: pointer; /* Cursor on hover */
        border-radius: 0.3rem;
    }

    .slider::-moz-range-thumb {
        width: 25px; /* Set a specific slider handle width */
        height: 25px; /* Slider handle height */
        background: #04aa6d; /* Green background */
        cursor: pointer; /* Cursor on hover */
    }

	.output {
        /* font style */
        font-family: "Roboto", sans-serif;
        font-weight: bold;
		/* padding-bottom: 10px; */
		padding-left: 10px;
		padding-right: 10px;
		min-width: 140px;
    }


	:global(svg g path),
	:global(svg g line) {
		fill: none;
		stroke: black;
		stroke-width: 1.3px; /* Make the axis lines thicker */
	}

	:global(svg g text) {
		font-size: 15px; /* Make the text bigger */
	}
</style>
