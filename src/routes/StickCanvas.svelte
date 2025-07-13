<script lang="ts">
	type Point = {
		x: number;
		y: number;
	};

	const DEBUG = false;

	export let stickData;

	let canvas: HTMLCanvasElement;

	// Redraw whenever any prop/store changes
	$: !DEBUG && canvas && draw($stickData);

	function draw(data: any) {
		if (!canvas) return;
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		const size = canvas.width;
		const cx = size / 2,
			cy = size / 2,
			maxR = (size / 2) * 0.9;

		ctx.clearRect(0, 0, size, size);

		// draw a blue square in the center
		ctx.fillStyle = '#9e9d9d';
		ctx.fillRect(cx - maxR, cy - maxR, maxR * 2, maxR * 2);

		// 1) Raw outline from calibCurve
		const rawPts: Point[] = data.calibCurve.map((c: number, i: number) => {
			const ang = (i * (360 / data.calibCurve.length) * Math.PI) / 180;
			const r = (c / 100) * maxR;
			// draw anticlockwise
			return { x: cx + Math.cos(-ang) * r, y: cy + Math.sin(-ang) * r };
		});

		DEBUG && console.log('calibCurve:', data.calibCurve);
		DEBUG && console.log('rawPts:', rawPts);

		// Draw each point as a small circle for better visibility
		rawPts.forEach((p) => {
			ctx.beginPath();
			ctx.arc(p.x, p.y, 3, 0, 2 * Math.PI);
			ctx.fillStyle = p === rawPts[0] ? '#f55' : '#4366da';
			ctx.fill();
			ctx.closePath();
		});

		// 2) Dead-zone fill
		const dz = data.deadZone;
		ctx.setLineDash([]);
		ctx.fillStyle = 'rgba(100,100,100,0.3)';
		ctx.beginPath();
		rawPts.forEach((p, i) => {
			const px = cx + (p.x - cx) * dz;
			const py = cy + (p.y - cy) * dz;
			i === 0 ? ctx.moveTo(px, py) : ctx.lineTo(px, py);
		});
		ctx.closePath();
		ctx.fill();

		// 3) Gate-size octagon
		ctx.strokeStyle = '#f55';
		ctx.lineWidth = 2;
		const gateR = data.gateSize * maxR;
		ctx.beginPath();
		for (let i = 0; i < 8; i++) {
			const ang = (i * 45 * Math.PI) / 180;
			const px = cx + Math.cos(ang) * gateR;
			const py = cy + Math.sin(ang) * gateR;
			i === 0 ? ctx.moveTo(px, py) : ctx.lineTo(px, py);
		}
		ctx.closePath();
		ctx.stroke();

		// 4) Green dot = raw position
		const rx = data.rawX;
		const ry = data.rawY;

		const rRaw = Math.hypot(rx, ry);
		const aRaw = Math.atan2(ry, rx);
		const pxRaw = cx + Math.cos(aRaw) * rRaw * maxR;
		const pyRaw = cy + Math.sin(aRaw) * rRaw * maxR;
		ctx.fillStyle = '#5f5';
		ctx.beginPath();
		ctx.arc(pxRaw, pyRaw, 5, 0, 2 * Math.PI);
		ctx.fill();

		// 5) Red dot = calibrated (after dead-zone & gate)
		let rCal = Math.max(0, rRaw - dz) / (1 - dz);
		rCal = Math.min(rCal, 1);
		const pxCal = cx + Math.cos(aRaw) * rCal * gateR;
		const pyCal = cy + Math.sin(aRaw) * rCal * gateR;
		ctx.fillStyle = '#f55';
		ctx.beginPath();
		ctx.arc(pxCal, pyCal, 5, 0, 2 * Math.PI);
		ctx.fill();
	}
</script>

<canvas
	class="m-auto block h-[200px] w-[200px] bg-white"
	bind:this={canvas}
	width="200"
	height="200"
></canvas>

{#if DEBUG}
	<button on:click={() => draw($stickData)}> Redraw </button>
{/if}
