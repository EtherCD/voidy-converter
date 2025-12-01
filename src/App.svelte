<script lang="ts">
	import { FFmpeg } from "@ffmpeg/ffmpeg";
	import { fetchFile, toBlobURL } from "@ffmpeg/util";
	const baseURL = "https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm";

	let input: HTMLInputElement;
	let scale = $state(0);
	let img: HTMLImageElement;
	let ffmpeg = $state(new FFmpeg());
	let loaded = $state(false);
	let cutTime = $state(false);
	let startTime = $state(0);
	let endTime = $state(0);
	let fps = $state(15);
	let empty = $state(true);

	const load = async () => {
		ffmpeg.on("log", (msg) => console.log(msg));
		ffmpeg.on("progress", (e) => console.error(e));
		await ffmpeg.load({
			coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
			wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, "application/wasm"),
		});
		loaded = true;
		console.log("loaded");
	};

	const toGif = async () => {
		if (!loaded) return;
		if (!input.files?.length) return;

		const file = input.files[0];
		const ext = file.name.split(".").pop();
		const fileName = `input.${ext}`;

		await ffmpeg.writeFile(fileName, await fetchFile(file));
		await ffmpeg.writeFile("palette.png", "");

		await ffmpeg.exec(["-i", fileName, "-vf", `fps=${fps},scale=${scale}:-1:flags=lanczos,palettegen`, "palette.png"]);
		let args = ["-i", fileName, "-i", "palette.png"];

		if (cutTime) args.push("-ss", `${Math.floor(startTime / 60)}:0${startTime % 60}00`, "-to", `${Math.floor(endTime / 60)}:0${endTime % 60}00`);

		args.push("-filter_complex", `fps=${fps},scale=${scale}:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5`, "t.gif");
		await ffmpeg.exec(args);

		const data = await ffmpeg.readFile("t.gif");
		img.src = URL.createObjectURL(new Blob([data as any], { type: "image/gif" }));

		empty = false;
	};

	$effect(() => {
		if (!loaded) load();
	});
</script>

<main class={"flex justify-center items-center w-[100vw] h-[100vh]"}>
	<div class={"p-1 outline-2 rounded-2xl flex flex-row gap-0.5 "}>
		<div class={"p-1 outline-2 rounded-2xl flex flex-col gap-0.5 items-start"}>
			<!-- <div>
			<h1 class={"text-3xl"}>You entering real void</h1>
			<h1 class={"text-2xl opacity-50"}>Links to my websites/social</h1>
		</div>
		<a href="https://ether.cd" class="bg-[#454d62] outline-1 p-0.5 rounded-2xl flex gap-0.5"> <svg width={16} height={16}><use href={"/link.svg"} /></svg>My website</a> -->
			<h1 class={"text-2xl"}>Online video transform to gif, video cutter</h1>
			<div class={"flex gap-1"}>
				<div class={"flex flex-col w-10 gap-0.5"}>
					<label for="#file" class={"text-xl"}>File</label>
					<input type="file" id={"file"} bind:this={input} class={"bg-[#454d62]  p-1 rounded-2xl"} accept="video/*" />
					<label for="#scale" class={"text-xl m-0"}>Resolution</label>
					<input type="number" name="Scale" id={"scale"} bind:value={scale} class={"bg-[#454d62] p-1 rounded-2xl"} />
					<label for="#fps" class={"text-xl m-0"}>Resolution</label>
					<input type="number" name="FPS" id={"fps"} bind:value={fps} class={"bg-[#454d62] p-1 rounded-2xl"} />
				</div>
				<div class={"flex flex-col w-10 gap-0.5"}>
					<label for="#startTime" class={"text-xl m-0"}>Start Time</label>
					<input type="number" name="Scale" disabled={!cutTime} id={"startTime"} step={0.1} bind:value={startTime} class={"bg-[#454d62] p-1 rounded-2xl disabled:opacity-50"} />
					<label for="#endTime" class={"text-xl m-0"}>End Time</label>
					<input type="number" name="Scale" disabled={!cutTime} id={"endTime"} bind:value={endTime} step={0.1} class={"bg-[#454d62] p-1 rounded-2xl disabled:opacity-50"} />

					<div>
						<label for="#check" class={"text-xl m-0"}>Enable time cut</label>
						<input type="checkbox" id={"check"} class={"bg-[#454d62] p-1 rounded-2xl disabled:opacity-50"} bind:checked={cutTime} />
					</div>
				</div>
			</div>
		</div>
		<div class={"bg-[#454d62] rounded-2xl p-1 flex flex-col gap-0.5 text-center"}>
			<h1>Output image</h1>
			<img src="/favicon.svg" alt="" bind:this={img} width={200} height={200} />
			<button onclick={toGif} class={"bg-[#2c2332] p-1 rounded-2xl"}>Go!</button>
		</div>
	</div>
</main>
