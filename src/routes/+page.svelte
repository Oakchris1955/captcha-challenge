<script lang="ts">
	import { onMount } from 'svelte';
	import type { Writable } from 'svelte/store';
	import { writable } from 'svelte/store';

	interface Guesses {
		correct: number;
		wrong: number;
		skipped: number;
	}

	let guesses: Writable<Guesses | undefined> = writable();

	guesses.subscribe((value) => {
		if (value !== undefined) {
			window.localStorage.setItem('guesses', JSON.stringify(value));
		}
	});

	let dialog: HTMLDialogElement;

	let canvas: HTMLCanvasElement;
	let displayedString = randomString();
	let inputString: string = '';

	const maxInitialOffset = 35;
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
		let temp_metrics = ctx.measureText(text) as TextMetrics & { height: number };
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

		// Distort the canvas
		ctx.setTransform(
			1,
			0,
			Math.max(0.25, Math.random() - 0.5) * (Math.random() < 0.5 ? 1 : -1),
			1,
			0,
			Math.max(0.25, Math.random() - 0.5) * (Math.random() < 0.5 ? 1 : -1)
		);

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

		// Draw a Bézier curve in the canvas (may or may not be drawn over text)
		ctx.beginPath();
		ctx.moveTo(
			(canvas.width / 3) * Math.random(),
			Math.max(canvas.height / 3, (canvas.height / 2) * Math.random())
		);
		ctx.bezierCurveTo(
			canvas.width * Math.random(),
			canvas.height * Math.random(),
			canvas.width * Math.random(),
			canvas.height * Math.random(),
			(canvas.width / 4) * Math.max(3, 4 * Math.random()),
			(canvas.height / 4) * Math.max(3, 4 * Math.random())
		);
		ctx.lineWidth = 2;
		ctx.stroke();

		// Remove distortion effect from future canvas actions (doesn't impact what has already been drawn)
		ctx.resetTransform();
	}

	function loadCanvas() {
		const ctx = canvas.getContext('2d');

		if (ctx !== null) {
			if (canvas.width !== canvas.clientWidth || canvas.height !== canvas.clientHeight) {
				canvas.width = canvas.clientWidth;
				canvas.height = canvas.clientHeight;

				console.debug(`Canvas width: ${canvas.clientWidth}px, height: ${canvas.clientHeight}px`);
			}

			drawObfuscatedText(ctx, displayedString);

			// Print canvas text to console only in dev mode (https://stackoverflow.com/a/71601719/)
			if (import.meta.env.DEV) {
				console.debug(`Displayed text: ${displayedString}`);
			}
		}
	}

	/** Load guess stats from `localStorage` */
	function loadGuessStats() {
		let localGuesses = JSON.parse(window.localStorage.getItem('guesses') ?? '{}');

		$guesses = {
			correct: +localGuesses.correct || 0,
			wrong: +localGuesses.wrong || 0,
			skipped: +localGuesses.skipped || 0
		};
	}

	onMount(() => {
		loadGuessStats();
		loadCanvas();
	});
</script>

<button id="settingsButton" on:click={() => dialog.showModal()}>
	<svg xmlns="http://www.w3.org/2000/svg" id="settingsIcon" viewBox="0 0 512 512"
		><!--! Font Awesome Free 6.4.2 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license (Commercial License) Copyright 2023 Fonticons, Inc. --><path
			d="M495.9 166.6c3.2 8.7 .5 18.4-6.4 24.6l-43.3 39.4c1.1 8.3 1.7 16.8 1.7 25.4s-.6 17.1-1.7 25.4l43.3 39.4c6.9 6.2 9.6 15.9 6.4 24.6c-4.4 11.9-9.7 23.3-15.8 34.3l-4.7 8.1c-6.6 11-14 21.4-22.1 31.2c-5.9 7.2-15.7 9.6-24.5 6.8l-55.7-17.7c-13.4 10.3-28.2 18.9-44 25.4l-12.5 57.1c-2 9.1-9 16.3-18.2 17.8c-13.8 2.3-28 3.5-42.5 3.5s-28.7-1.2-42.5-3.5c-9.2-1.5-16.2-8.7-18.2-17.8l-12.5-57.1c-15.8-6.5-30.6-15.1-44-25.4L83.1 425.9c-8.8 2.8-18.6 .3-24.5-6.8c-8.1-9.8-15.5-20.2-22.1-31.2l-4.7-8.1c-6.1-11-11.4-22.4-15.8-34.3c-3.2-8.7-.5-18.4 6.4-24.6l43.3-39.4C64.6 273.1 64 264.6 64 256s.6-17.1 1.7-25.4L22.4 191.2c-6.9-6.2-9.6-15.9-6.4-24.6c4.4-11.9 9.7-23.3 15.8-34.3l4.7-8.1c6.6-11 14-21.4 22.1-31.2c5.9-7.2 15.7-9.6 24.5-6.8l55.7 17.7c13.4-10.3 28.2-18.9 44-25.4l12.5-57.1c2-9.1 9-16.3 18.2-17.8C227.3 1.2 241.5 0 256 0s28.7 1.2 42.5 3.5c9.2 1.5 16.2 8.7 18.2 17.8l12.5 57.1c15.8 6.5 30.6 15.1 44 25.4l55.7-17.7c8.8-2.8 18.6-.3 24.5 6.8c8.1 9.8 15.5 20.2 22.1 31.2l4.7 8.1c6.1 11 11.4 22.4 15.8 34.3zM256 336a80 80 0 1 0 0-160 80 80 0 1 0 0 160z"
		/></svg
	></button
>

<!-- svelte-ignore a11y-click-events-have-key-events a11y-no-noninteractive-element-interactions -->
<dialog bind:this={dialog} on:click|self={() => dialog.close()}>
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div on:click|stopPropagation>
		<button
			on:click={() => {
				if (confirm('Are you sure?')) {
					window.localStorage.removeItem('guesses');
					loadGuessStats();
				}

				dialog.close();
			}}>Clear guess stats</button
		>

		<!-- svelte-ignore a11y-autofocus -->
		<button
			autofocus
			on:click={() => {
				dialog.close();
			}}>Close</button
		>
	</div>
</dialog>

<!-- Wait for "guesses" object variable to be initialized before displaying this -->
<div id="counterContainer" class="column">
	<div>
		<div class="left">Correct guesses: {$guesses?.correct ?? ''}</div>
		<div class="right">Wrong guesses: {$guesses?.wrong ?? ''}</div>
	</div>

	Skipped: {$guesses?.skipped ?? ''}
</div>

<div id="canvasContainer">
	<canvas bind:this={canvas} />

	<button
		id="regenerateButton"
		on:click={() => {
			if ($guesses !== undefined) {
				$guesses.skipped += 1;
				inputString = '';
				displayedString = randomString();
				loadCanvas();
			}
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

		if ($guesses !== undefined) {
			let trimmedInputString = inputString.trim();

			if (trimmedInputString.length !== 0) {
				if (trimmedInputString === displayedString) {
					$guesses.correct += 1;
				} else {
					$guesses.wrong += 1;
				}

				inputString = '';
				displayedString = randomString();
				loadCanvas();
			}
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
	button,
	input[type='submit'] {
		background-color: rgb(36, 48, 94);
		color: rgb(228, 233, 242);
		border: none;
		border-radius: 5px;

		margin: 2px;
		padding: 5px 8px;

		font-size: 0.9rem;
	}

	button:hover,
	input[type='submit']:hover {
		cursor: pointer;
	}

	button:active,
	input[type='submit']:active {
		cursor: default;
	}

	:global(body) {
		background-color: rgb(22, 29, 59);
		color: white;
	}

	:global(html, body) {
		height: 100%;
		margin: 0;
	}

	:global(#container),
	:global(#container) *:not(dialog) {
		display: flex;
		justify-content: center;
		align-items: center;
	}

	:global(#container) {
		flex-direction: column;
		gap: 1%;
	}

	.column {
		flex-direction: column;
	}

	.left {
		align-self: flex-start;
		text-align: left;
	}

	.right {
		align-self: flex-end;
		text-align: right;
	}

	canvas {
		aspect-ratio: 3/1;
		image-rendering: pixelated;
		width: 250px;

		background-color: white;
	}

	dialog {
		/*background-color: rgb(27, 37, 78);*/
		background-color: rgb(17, 30, 80);

		border: none;
		border-radius: 5px;
		padding: 0;
	}

	dialog > div {
		flex-direction: column;

		gap: 3px;
		padding: 1em;
	}

	dialog::backdrop {
		background: rgba(0, 0.3, 0, 0.3);
		width: 100%;
	}

	#settingsButton {
		height: 5%;
		position: absolute;
		top: 2.5%;
		right: 2.5%;

		background-color: transparent;
		border: 0;
		padding: 5px;
	}

	#settingsIcon {
		height: 100%;
		width: 100%;
	}

	#counterContainer {
		width: 300px;
		justify-content: space-between;
		gap: 5px;
	}

	#counterContainer * {
		width: 100%;
	}

	#reloadIcon {
		height: 70%;
	}

	#regenerateButton {
		background-color: rgb(23, 45, 87);

		height: 100%;
		aspect-ratio: 1/1;
		border: 0;
		border-top-right-radius: 20px;

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
