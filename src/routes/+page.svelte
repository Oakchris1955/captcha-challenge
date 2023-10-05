<script lang="ts">
	import { onMount } from 'svelte';

	let canvas: HTMLCanvasElement;
	let displayedString = randomString();
	let inputString: string;

	const maxInitialOffset = 25;
	const maxVerticalChange = 20;
	const maxLetterAngle = 45;
	const minLetterAngle = 15;

	function randomString(): string {
		return (Math.random() + 1).toString(36).substring(5);
	}

	function measureMetrics(
		ctx: CanvasRenderingContext2D,
		text: string
	): TextMetrics & { height: number } {
		let temp_metrics: any = ctx.measureText(text);
		temp_metrics.height =
			temp_metrics.actualBoundingBoxAscent + temp_metrics.actualBoundingBoxDescent;

		return temp_metrics;
	}

	function drawText(
		ctx: CanvasRenderingContext2D,
		text: string,
		x: number,
		y: number,
		degrees: number = 0
	) {
		// Convert degrees to radians
		const radians = degrees * (Math.PI / 180);
		const metrics = measureMetrics(ctx, text);

		// Offset canvas by the x and y input coordinates to draw text
		ctx.translate(x, y);
		// Rotate canvas plane
		ctx.rotate(radians);
		// Draw text centered at x and y coordinates
		ctx.fillText(text, -metrics.width / 2, metrics.height / 2);
		// Undo canvas rotation and offset
		ctx.rotate(-radians);
		ctx.translate(-x, -y);
	}

	function drawObfuscatedText(ctx: CanvasRenderingContext2D, text: string) {
		ctx.fillStyle = 'white';
		ctx.fillRect(0, 0, canvas.width, canvas.height);

		ctx.fillStyle = 'black';
		ctx.font = `${canvas.width / 6}px serif`;

		let width_offset = Math.random() * maxInitialOffset;
		for (const char of text) {
			width_offset += measureMetrics(ctx, char).width * Math.max(Math.random() * 1.5, 0.75);

			drawText(
				ctx,
				char,
				width_offset,
				canvas.height / 2 + Math.random() * maxVerticalChange,
				Math.max(
					(Math.random() - 0.5) * maxLetterAngle * 2,
					Math.random() < 0.5 ? 1 : -1 * minLetterAngle
				)
			);
		}
	}

	function loadCanvas() {
		const ctx = canvas.getContext('2d');

		if (ctx !== null) {
			if (canvas.width !== canvas.clientWidth || canvas.height !== canvas.clientHeight) {
				canvas.width = canvas.clientWidth * window.devicePixelRatio;
				canvas.height = canvas.clientHeight * window.devicePixelRatio;

				console.debug(`Canvas width: ${canvas.clientWidth}px, height: ${canvas.clientHeight}px`);
			}

			drawObfuscatedText(ctx, displayedString);

			console.debug(`Displayed text: ${displayedString}`);
		}
	}

	onMount(() => {
		loadCanvas();
	});
</script>

<div id="canvasContainer">
	<canvas bind:this={canvas} />

	<button
		id="regenerateButton"
		on:click={() => {
			displayedString = randomString();
			loadCanvas();
		}}
	>
		<svg xmlns="http://www.w3.org/2000/svg" id="reloadIcon" viewBox="0 0 512 512"
			><!--! Font Awesome Free 6.4.2 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license (Commercial License) Copyright 2023 Fonticons, Inc. --><path
				fill="rgb(24, 61, 114)"
				d="M463.5 224H472c13.3 0 24-10.7 24-24V72c0-9.7-5.8-18.5-14.8-22.2s-19.3-1.7-26.2 5.2L413.4 96.6c-87.6-86.5-228.7-86.2-315.8 1c-87.5 87.5-87.5 229.3 0 316.8s229.3 87.5 316.8 0c12.5-12.5 12.5-32.8 0-45.3s-32.8-12.5-45.3 0c-62.5 62.5-163.8 62.5-226.3 0s-62.5-163.8 0-226.3c62.2-62.2 162.7-62.5 225.3-1L327 183c-6.9 6.9-8.9 17.2-5.2 26.2s12.5 14.8 22.2 14.8H463.5z"
			/></svg
		></button
	>
</div>

<form
	id="submitForm"
	on:submit={(e) => {
		e.preventDefault();

		if (inputString.trim() === displayedString) {
			alert('Correct');
		} else {
			alert('Not correct');
		}
	}}
>
	<label for="obfuscatedText">Obfuscated text: </label>
	<input
		type="text"
		id="obfuscatedText"
		name="obfuscatedText"
		bind:value={inputString}
		autocomplete="off"
	/>
	<input type="submit" />
</form>

<style>
	button:hover {
		cursor: pointer;
	}

	button:active {
		cursor: default;
	}

	:global(body) {
		background-color: black;
		color: white;
	}

	:global(html, body) {
		height: 100%;
		margin: 0;
	}

	:global(#container),
	:global(#container) * {
		display: flex;
		justify-content: center;
		align-items: center;
	}

	:global(#container) {
		flex-direction: column;
		gap: 1%;
	}

	canvas {
		aspect-ratio: 3/1;
		image-rendering: pixelated;
		width: 250px;

		background-color: white;
	}

	#reloadIcon {
		height: 70%;
	}

	#regenerateButton {
		background-color: transparent;

		height: 100%;
		aspect-ratio: 1/1;
		border: 0;

		font-family: Lucida Sans Unicode;
	}

	#submitForm {
		flex-direction: column;
		gap: 2px;
	}

	#submitForm label {
		align-self: flex-start;
		margin-left: 2%;
	}

	#submitForm input[type='submit'] {
		width: 50%;
	}
</style>
