<script lang="ts">
	interface Props {
		min: number;
		max: number;
		onchange: (min: number, max: number) => void;
	}

	let rangeTrack: HTMLDivElement;
	let rangeMin: HTMLInputElement;
	let rangeMax: HTMLInputElement;
	let minValue: HTMLSpanElement;
	let maxValue: HTMLSpanElement;

	function update() {
		let min = parseFloat(rangeMin.value);
		let max = parseFloat(rangeMax.value);

		min = parseFloat(rangeMin.value);
		max = parseFloat(rangeMax.value);

		const percentMin = (min / parseFloat(rangeMin.max)) * 100;
		const percentMax = (max / parseFloat(rangeMax.max)) * 100;

		rangeTrack.style.left = percentMin + "%";
		rangeTrack.style.right = 100 - percentMax + "%";

		minValue.textContent = min + "";
		maxValue.textContent = max + "";

		props.onchange(min, max);
	}

	$effect(() => {
		rangeMin.addEventListener("input", update);
		rangeMax.addEventListener("input", update);
		rangeMax.value = props.max + "";
		maxValue.textContent = props.max + "";
		minValue.textContent = "0";
		rangeMin.value = "0";
		update();
	});

	let props: Props = $props();
</script>

<div class="w-full max-w-xl">
	<div class="relative h-2 bg-(--bg) rounded-2xl">
		<div bind:this={rangeTrack} class="absolute h-2 bg-[#93bbba] rounded-2xl"></div>

		<input bind:this={rangeMin} type="range" min={props.min} max={props.max} step={0.01} class="absolute w-full h-2 bg-transparent appearance-none pointer-events-none ml-[-50%]" />

		<input bind:this={rangeMax} type="range" min={props.min} max={props.max} step={0.01} class="absolute w-full h-2 bg-transparent appearance-none pointer-events-none ml-[-50%]" />
	</div>

	<div class="flex justify-between text-lg">
		<span bind:this={minValue} class="font-mono">0</span>
		<span bind:this={maxValue} class="font-mono">0</span>
	</div>
</div>

<style>
	input[type="range"]::-webkit-slider-thumb {
		pointer-events: all;
		width: 18px;
		height: 18px;
		border-radius: 9999px;
		background: white;
		border: 2px solid #2563eb;
		cursor: pointer;
		appearance: none;
	}

	input[type="range"]::-moz-range-thumb {
		pointer-events: all;
		width: 18px;
		height: 18px;
		border-radius: 9999px;
		background: white;
		border: 2px solid #2563eb;
		cursor: pointer;
	}

	input[type="range"]::-webkit-slider-runnable-track {
		background: transparent;
	}
</style>
