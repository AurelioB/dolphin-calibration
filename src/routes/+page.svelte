<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	import Stick from './Stick.svelte';

	let gamepadName = '';

	const left = writable({
		calibrating: false,
		gamepadIndex: -1,
		id: 'Main Stick',
		rawX: 0,
		rawY: 0,
		deadZone: 0.1,
		gateSize: 0.7937,
		calibCurve: [],
		calibString: '',
		samples: [],
		axes: [0, 1]
	});

	const right = writable({
		calibrating: false,
		gamepadIndex: -1,
		id: 'C-Stick',
		rawX: 0,
		rawY: 0,
		deadZone: 0.1,
		gateSize: 0.7221,
		calibCurve: [],
		calibString: '',
		samples: [],
		axes: [2, 3]
	});

	onMount(() => {
		// Check for already-connected gamepads
		const gps = navigator.getGamepads();
		for (let i = 0; i < gps.length; i++) {
			if (gps[i]) {
				$left.gamepadIndex = $right.gamepadIndex = i;
				gamepadName = gps[i]!.id;
				break;
			}
		}

		window.addEventListener('gamepadconnected', (e: GamepadEvent) => {
			$left.gamepadIndex = $right.gamepadIndex = e.gamepad.index;
			gamepadName = e.gamepad.id;
		});

		window.addEventListener('gamepaddisconnected', (e: GamepadEvent) => {
			gamepadName = 'No controller';
			$left.gamepadIndex = $right.gamepadIndex = -1;
		});
	});

	const buildConfigString = (sticks: any[]) =>
		sticks
			.map((stick) =>
				[
					`${stick.id}/Calibration = ${stick.calibString}`,
					`${stick.id}/Dead Zone = ${(stick.deadZone * 100).toFixed(2)}`,
					`${stick.id}/Gate Size = ${(stick.gateSize * 100).toFixed(2)}`
				].join('\n')
			)
			.join('\n');

	const copyToClipboard = (text: string) => {
		navigator.clipboard.writeText(text);
	};
</script>

<div class="p-[30px]">
	<div class="top">
		<h1>Stick Calibration</h1>
		<div>Controller: {gamepadName}</div>
	</div>

	<div class="container">
		<div class="stick-panel">
			<Stick stickData={left}></Stick>
		</div>

		<div class="stick-panel">
			<Stick stickData={right}></Stick>
		</div>
	</div>
	<div
		class="mt-[10px] cursor-pointer bg-gray-300 p-[5px] font-mono whitespace-pre"
		on:click={() => copyToClipboard(buildConfigString([$left, $right]))}
	>
		{buildConfigString([$left, $right])}
	</div>
</div>

<style>
	h1 {
		margin-bottom: 0.5rem;
	}
	.top {
		margin-bottom: 1rem;
	}
	.container {
		display: flex;
		gap: 2rem;
	}
	.stick-panel {
		width: 240px;
	}
</style>
