<script lang="ts">
	import StickCanvas from './StickCanvas.svelte';

	export let stickData;

	// Helpers for binning & normalization
	function computeBins(s: { x: number; y: number }[], size = 16) {
		const bins = Array(size).fill(0);
		for (const { x, y } of s) {
			const r = Math.hypot(x, y);
			const deg = ((Math.atan2(y, x) * 180) / Math.PI + 360) % 360;
			const idx = Math.floor(deg / (360 / size)) % size;
			bins[idx] = Math.max(bins[idx], r);
		}
		return bins;
	}

	function normalizeByCardinal(b: number[]) {
		const avg = (b[0] + b[4] + b[8] + b[12]) / 4;
		if (avg === 0) return b.map(() => 100);
		return b.map((v) => ((v / avg) * 100).toFixed(2));
	}

	let gamepadStarted = false;
	function pollLoop(idx: number) {
		const gp = navigator.getGamepads()[idx];
		if (gp) {
			const [x, y] = [gp.axes[$stickData.axes[0]], gp.axes[$stickData.axes[1]]];

			$stickData.rawX = x;
			$stickData.rawY = y;

			if ($stickData.calibrating) {
				$stickData.samples.push({ x, y });

				$stickData.calibCurve = normalizeByCardinal(computeBins($stickData.samples, 32));
				$stickData.calibCurve.reverse();
				$stickData.calibString = $stickData.calibCurve.join(' ');
			}
		}
		requestAnimationFrame(() => pollLoop(idx));
	}

	const start = (idx: number) => {
		if (!gamepadStarted && idx > -1) {
			console.log(`Starting gamepad polling for stick: ${$stickData.id} at index ${idx}`);
			gamepadStarted = true;
			pollLoop(idx);
		}
	};

	$: start($stickData.gamepadIndex);

	// Parse manual edits to the calibration arrays
	function onTextareaChange(v: string) {
		if ($stickData.calibrating) return;
		const nums = v
			.trim()
			.split(/\s+/)
			.map((s) => parseFloat(s))
			.filter((n) => !isNaN(n));
		if (nums.length > 2) {
			$stickData.calibCurve = nums;
		}
	}
</script>

<h3>{$stickData.id}</h3>

<div>
	<button
		disabled={$stickData.gamepadIndex < 0}
		class="rounded bg-gray-300 px-1"
		class:bg-green-300={$stickData.gamepadIndex > -1 && !$stickData.calibrating}
		class:bg-red-500={$stickData.gamepadIndex > -1 && $stickData.calibrating}
		on:click={() => {
			$stickData.calibrating = !$stickData.calibrating;
			if ($stickData.calibrating) {
				$stickData.samples = [];
			}
		}}
	>
		{#if $stickData.calibrating}
			Stop Calibration
		{:else}
			Start Calibration
		{/if}
	</button>
</div>

<label>
	Dead Zone: {($stickData.deadZone * 100).toFixed(0)}%
	<input type="range" min="0" max="0.5" step="0.01" bind:value={$stickData.deadZone} />
</label>

<label>
	Gate Size: {($stickData.gateSize * 100).toFixed(2)}%
	<input type="range" min="0" max="1" step="0.0001" bind:value={$stickData.gateSize} />
</label>

<label>
	Calibration values:
	<textarea bind:value={$stickData.calibString}></textarea>
</label>

<button on:click={() => onTextareaChange($stickData.calibString)}>Update</button>

<StickCanvas {stickData} />

<!--<div>-->
<!--	{#each $stickData.calibCurve as c, i}-->
<!--		<div class="">{i}: {c?.toFixed?.(2)}</div>-->
<!--	{/each}-->
<!--</div>-->

<style>
	label {
		display: block;
		margin: 0.5rem 0;
		font-size: 0.9rem;
	}
	textarea {
		width: 100%;
		height: 4rem;
		font-family: monospace;
	}
	input[type='range'] {
		width: 100%;
	}
	button {
		margin-bottom: 1rem;
	}
</style>
