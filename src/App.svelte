<script lang="ts">
	import { FFmpeg } from "@ffmpeg/ffmpeg";
	import { fetchFile, toBlobURL } from "@ffmpeg/util";
	import DragAFile from "./lib/dragafile.svelte";
	import Window from "./lib/window.svelte";
	const baseURL = "https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm";

	let file: File | undefined = $state();
	let ffmpegLoaded = $state(false);

	let scale = $state(0);

	let isTimeCut = $state(false);
	let timeCutStart = $state(0);
	let timeCutEnd = $state(0.1);

	let logs = $state(["Nothing here!"]);

	let fps = $state(15);
	let img: HTMLImageElement;
	let imageSrc = $state("");
	let imageName = $state("");
	let imageBlob: Blob | undefined = $state();

	let processed = $state(false);

	let ffmpeg = $state(new FFmpeg());
	const load = async () => {
		ffmpeg.on("log", (msg) => {
			if (logs.length > 100) logs = logs.slice(logs.length - 101, logs.length - 1);
			logs.push(`[${msg.type}]: ${msg.message}`);
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

		const ext = file.name.split(".").pop();
		const fileName = `input.${ext}`;

		await ffmpeg.writeFile(fileName, await fetchFile(file));
		await ffmpeg.writeFile("palette.png", "");

		await ffmpeg.exec(["-i", fileName, "-vf", `fps=${fps},scale=${scale}:-1:flags=lanczos,palettegen`, "palette.png"]);
		let args = ["-i", fileName, "-i", "palette.png"];

		if (isTimeCut) args.push("-ss", toTimeCode(timeCutStart), "-to", toTimeCode(timeCutEnd));

		args.push("-filter_complex", `fps=${fps},scale=${scale}:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5`, "t.gif");
		console.log(args);
		await ffmpeg.exec(args);

		const data = await ffmpeg.readFile("t.gif");
		imageBlob = new Blob([data as any], { type: "image/gif" });
		img.src = URL.createObjectURL(imageBlob);
		imageSrc = img.src;
		imageName = file.name + ".gif";
		processed = true;
	};

	const change = (event: Event) => {
		const target = event.target as HTMLInputElement;
		if (!target.files || target.files.length === 0) return;
		file = target.files[0];
		processed = false;
	};

	$effect(() => {
		if (!ffmpegLoaded) load();
	});
</script>

<main class={"flex justify-center md:items-center w-full h-[100vh] flex-col gap-1 p-1"}>
	{#if !file}
		<input type="file" id={"file"} class={"w-full h-full absolute opacity-0 pointer-none"} accept="video/*" onchange={change} />
		<DragAFile />
	{:else}
		<div class={"p-1 rounded-2xl outline-1 flex flex-col md:flex-row gap-1 mt-40 md:mt-0 "}>
			<Window class={"gap-1 flex flex-col"}
				><h1 class={"text-2xl"}>Selected File</h1>
				<video src={URL.createObjectURL(file)} class={"max-w-[300px]"} controls><track kind="captions" /></video></Window
			>
			<div>
				<h1 class={"text-2xl"}>Online video to gif converter</h1>
				<div class={"flex gap-0.5 flex-col md:flex-row "}>
					<div>
						<p class={"text-xl"}>Scale (width):</p>
						<input type="number" bind:value={scale} class={"p-1 bg-(--window-bg) rounded-2xl  md:w-10"} />
						<p class={"text-xl"}>FPS:</p>
						<input type="number" bind:value={fps} class={"p-1 bg-(--window-bg) rounded-2xl md:w-10"} />
					</div>

					<div class={"flex flex-col"}>
						<label for="#startTime" class={"text-xl m-0"}>Start Time</label>
						<input
							type="number"
							name="Scale"
							disabled={!isTimeCut}
							id={"startTime"}
							step={0.1}
							bind:value={timeCutStart}
							class={"bg-(--window-bg) p-1 rounded-2xl disabled:opacity-50 md:w-10"}
							min="0"
						/>
						<label for="#endTime" class={"text-xl m-0"}>End Time</label>
						<input
							type="number"
							name="Scale"
							disabled={!isTimeCut}
							id={"endTime"}
							bind:value={timeCutEnd}
							step={0.1}
							class={"bg-(--window-bg) p-1 rounded-2xl disabled:opacity-50 md:w-10"}
							min="0.1"
						/>

						<div>
							<label for="#check" class={"text-xl m-0"}>Enable time cut</label>
							<input type="checkbox" id={"check"} class={"bg-[#454d62] p-1 rounded-2xl disabled:opacity-50"} bind:checked={isTimeCut} />
						</div>
					</div>
				</div>
				<div class={"flex flex-col gap-0.5"}>
					<button class={"p-1 bg-(--window-bg) rounded-2xl w-full"} onclick={toGif}>Go!</button>
					<div class={"flex gap-0.5"}>
						<button
							class={"p-1 bg-(--window-bg) rounded-2xl w-full"}
							onclick={() => {
								file = undefined;
							}}>Select other video</button
						>

						{#if processed}
							<a class={"p-1 bg-(--window-bg) rounded-2xl w-full disabled:opacity-50 text-center"} href={imageSrc} download={imageName}>Download</a>
						{/if}
					</div>
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
		<div class={"p-1 rounded-2xl outline-1 flex flex-col gap-1"}>
			<h1>Logs</h1>
			<Window class={"text-left md:w-30  h-10 overflow-y-scroll"}>
				{#each logs as log}
					<p>{log}</p>
				{/each}
			</Window>
		</div>
	{/if}
</main>
