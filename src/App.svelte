<script lang="ts">
	import { FFmpeg } from "@ffmpeg/ffmpeg";
	import { fetchFile, toBlobURL } from "@ffmpeg/util";
	import DragAFile from "./lib/dragafile.svelte";
	import Window from "./lib/window.svelte";
	import Slider from "./lib/slider.svelte";
	import { tick } from "svelte";
	const baseURL = "https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm";

	let fileInput: HTMLInputElement;
	let file: File | undefined = $state();
	let fileName: string = $state("");
	let ffmpegLoaded = $state(false);

	let scale = $state(0);

	let cutTimeStart = $state(0);
	let cutTimeEnd = $state(0);

	let logs = $state(["Nothing here!"]);
	let logsContainer: HTMLDivElement;

	let fps = $state(15);
	let img: HTMLImageElement;
	let imageSrc = $state("");
	let imageName = $state("");
	let imageBlob: Blob | undefined = $state();
	let videoDuration = $state(0);

	let processed = $state(false);
	let inProcess = $state(false);
	let progress = $state(1);

	let ffmpeg = $state(new FFmpeg());
	const load = async () => {
		ffmpeg.on("log", (msg) => {
			if (logs.length > 20) logs = logs.slice(-20);
			logs.push(`[${msg.type}]: ${msg.message}`);
			tick().then(() => {
				logsContainer.scrollTop = logsContainer.scrollHeight;
			});
		});
		ffmpeg.on("progress", (prog) => {
			progress = prog.progress * 100;
		});
		await ffmpeg.load({
			coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
			wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, "application/wasm"),
		});
		ffmpegLoaded = true;
		console.log("loaded");
	};

	function toTimeCode(sec: number) {
		const h = Math.floor(sec / 3600)
			.toString()
			.padStart(2, "0");
		const m = Math.floor((sec % 3600) / 60)
			.toString()
			.padStart(2, "0");
		const s = Math.floor(sec % 60)
			.toString()
			.padStart(2, "0");
		if (Math.round(sec) != sec) {
			const a = Math.round(Math.round(sec * 10) - sec);
			return `${h}:${m}:${s}.${a}00`;
		}
		return `${h}:${m}:${s}`;
	}

	const toGif = async () => {
		if (!ffmpegLoaded || !file) {
			logs.push("FFMPEG is not loaded!");
			return;
		}
		if (inProcess) {
			return;
		}

		inProcess = true;
		const ext = file.name.split(".").pop();
		const fileName = `input.${ext}`;

		await ffmpeg.writeFile(fileName, await fetchFile(file));
		await ffmpeg.writeFile("palette.png", "");

		await ffmpeg.exec(["-i", fileName, "-vf", `fps=${fps},scale=${scale}:-1:flags=lanczos,palettegen`, "palette.png"]);
		let args = ["-i", fileName, "-i", "palette.png"];

		if (cutTimeStart !== 0 || cutTimeEnd !== videoDuration) args.push("-ss", toTimeCode(cutTimeStart), "-to", toTimeCode(cutTimeEnd));

		args.push("-filter_complex", `fps=${fps},scale=${scale}:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5`, "t.gif");
		console.log(args);
		await ffmpeg.exec(args);

		const data = await ffmpeg.readFile("t.gif");
		imageBlob = new Blob([data as any], { type: "image/gif" });
		img.src = URL.createObjectURL(imageBlob);
		imageSrc = img.src;
		imageName = file.name + ".gif";
		processed = true;

		inProcess = false;
	};

	const change = async (event: Event) => {
		const target = event.target as HTMLInputElement;
		if (!target.files || target.files.length === 0) return;

		try {
			if (file) await ffmpeg.deleteFile(fileName);
		} catch {}

		file = target.files[0];
		const ext = file.name.split(".").pop();
		fileName = `input.${ext}`;
		processed = false;

		cutTimeStart = 0;
		cutTimeEnd = videoDuration;
	};

	const onloadedmetadata = (event: Event) => {
		videoDuration = (event.target as HTMLVideoElement).duration;
		cutTimeStart = 0;
		cutTimeEnd = videoDuration;
	};

	const onchangeslider = (min: number, max: number) => {
		cutTimeStart = min;
		cutTimeEnd = max;
	};

	$effect(() => {
		if (!ffmpegLoaded) load();
	});
</script>

<main class={"flex justify-center md:items-center w-full min-h-screen flex-col gap-1 p-1"}>
	{#if !file}
		<input type="file" id={"file"} class={"w-full h-full absolute opacity-0 pointer-none pointer-none"} accept="video/*" onchange={change} bind:this={fileInput} />
		<!-- svelte-ignore a11y_consider_explicit_label -->
		<button
			class={"w-full h-full absolute "}
			onclick={() => {
				fileInput.click();
			}}
		></button>
		<DragAFile />
	{:else}
		<div class={"p-1 rounded-2xl outline-1 flex flex-col md:flex-row gap-0.5 "}>
			<Window class={"gap-1 flex flex-col"}>
				<h1 class={"text-2xl"}>Selected File</h1>
				<video src={URL.createObjectURL(file)} class={"max-w-[300px]"} controls {onloadedmetadata}><track kind="captions" /></video>
				<p class={"text-xl "}>Time cut</p>
				<Slider min={0} max={videoDuration} onchange={onchangeslider} />
			</Window>
			<div>
				<h1 class={"text-2xl"}>Online video to gif converter</h1>
				<div class={"flex gap-0.5 flex-col md:flex-row mb-0.5 "}>
					<div class={"w-full"}>
						<p class={"text-xl"}>Scale (width):</p>
						<input type="number" bind:value={scale} class={"p-1 bg-(--window-bg) rounded-2xl text-xl w-full"} />
						<p class={"text-xl "}>FPS:</p>
						<input type="number" bind:value={fps} class={"p-1 bg-(--window-bg) rounded-2xl text-xl w-full"} />
					</div>
				</div>
				<div class={"flex flex-col gap-0.5"}>
					<button class={"p-1 bg-(--window-bg) rounded-2xl w-full text-xl"} onclick={toGif}>Go!</button>
					<div class={"flex gap-0.5"}>
						<button
							class={"p-1 bg-(--window-bg) rounded-2xl w-full text-xl"}
							onclick={() => {
								file = undefined;
							}}>Select other video</button
						>

						{#if processed}
							<a class={"p-1 bg-(--window-bg) rounded-2xl w-full disabled:opacity-50 text-center"} href={imageSrc} download={imageName}>Download</a>
						{/if}
					</div>

					<progress
						class={`"appearance-none w-full h-2 overflow-hidden rounded-xl 
           bg-(--window-bg)
           [&::-webkit-progress-bar]:bg-(--window-bg)
           [&::-webkit-progress-value]:bg-(--color)
           [&::-moz-progress-bar]:bg-(--color)`}
						value={progress}
						max={100}
					></progress>
				</div>
			</div>

			<Window class={"gap-1 flex flex-col"}
				><h1 class={"text-2xl"}>Output image</h1>
				<img bind:this={img} class={"max-w-[300px]"} src={"/favicon.svg"} alt={""} />
				{#if imageBlob}
					<p>Image size: {(imageBlob.size / 1024 / 1024).toFixed(2)} MiB</p>
				{/if}
			</Window>
		</div>
		<div class={"p-1 rounded-2xl outline-1 flex flex-col gap-1 text-xl"}>
			<h1>Logs</h1>
			<div class={"text-left md:w-30  h-10 overflow-y-scroll"} bind:this={logsContainer}>
				<Window class={"text-left"}>
					{#each logs as log}
						<p>{log}</p>
					{/each}
				</Window>
			</div>
		</div>
	{/if}
</main>
