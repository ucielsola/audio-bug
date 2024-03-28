<script>
	import { writable } from "svelte/store";

	const data = writable({
		method: "linear",
		duration: undefined,
		currentTime: undefined,
		playing: false,
		gain: undefined,
		pauseRequested: false,
	});

	let showButtons = false;

	let context;
	let audio;
	let source;
	let leftGainNode;
	let rightGainNode;
	let output;

	async function loadTrack(trackUrl) {
		const response = await fetch(trackUrl);

		context = new AudioContext({ latencyHint: 0.3 });
		audio = new Audio(trackUrl);
		audio.crossOrigin = "anonymous";
		source = context.createMediaElementSource(audio);

		const splitter = context.createChannelSplitter(2);

		source.connect(splitter);

		leftGainNode = context.createGain();
		rightGainNode = context.createGain();

		splitter.connect(leftGainNode, 0, 0);
		splitter.connect(rightGainNode, 1, 0);

		const merger = context.createChannelMerger(2);

		leftGainNode.connect(merger, 0, 0);
		rightGainNode.connect(merger, 0, 1);

		output = new GainNode(context, { gain: 1 });

		merger.connect(output);
		output.connect(context.destination);

		audio.addEventListener("canplaythrough", () => {
			showButtons = true;
		});

		output.gain.value = 0;
	}

	function play() {
		data.update((d) => ({ ...d, playing: true }));

		switch ($data.method) {
			case "linear":
				linearFadeIn();
				break;
			case "exponential":
				exponentialFadeIn();
				break;
			default:
				break;
		}

		audio.play();
		updateData();
	}

	function pause() {
		data.update((d) => ({ ...d, pauseRequested: true }));

		switch ($data.method) {
			case "linear":
				linearFadeOut();
				break;
			case "exponential":
				exponentialFadeOut();
				break;
			default:
				break;
		}
	}

	function linearFadeOut() {
		let start = context.currentTime;
		let end = start + 5;
		const currentGain = output.gain.value;

		output.gain.linearRampToValueAtTime(currentGain, start);
		output.gain.linearRampToValueAtTime(0.01, end);
	}

	function linearFadeIn() {
		let start = context.currentTime;
		let end = start + 5;

		output.gain.linearRampToValueAtTime(0.1, start);
		output.gain.linearRampToValueAtTime(1, end);
	}

	function exponentialFadeIn() {
		let start = context.currentTime;
		let end = start + 5;

		output.gain.exponentialRampToValueAtTime(0, start);
		output.gain.exponentialRampToValueAtTime(1, end);
	}

	function exponentialFadeOut() {
		let start = context.currentTime;
		let end = start + 5;
		const currentGain = output.gain.value;

		output.gain.exponentialRampToValueAtTime(currentGain, start);
		output.gain.exponentialRampToValueAtTime(0.01, end);
	}

	function updateData() {
		if (!audio) return;

		data.update((state) => ({
			...state,
			playing: !audio.paused,
			duration: audio.duration,
			currentTime: audio.currentTime,
			gain: output.gain.value,
		}));

		if ($data.playing) {
			requestAnimationFrame(updateData.bind(this));
		}
	}

	$: if ($data.pauseRequested && $data.gain <= 0.01) {
		audio.pause();
		data.update((d) => ({ ...d, pauseRequested: false }));
	}
</script>

<h1>Audio Bug tests</h1>
<div class="row">
	<button on:click={() => loadTrack("/vibro-music.m4a")}>
		Load vibro + music
	</button>
	<button on:click={() => loadTrack("/vibro.m4a")}>Load vibro only</button>
</div>
<div class="row">
	<fieldset
		on:change={(x) => data.update((s) => ({ ...s, method: x.target.name }))}
	>
		<legend>Fade type:</legend>

		<div>
			<input
				type="checkbox"
				id="linear"
				name="linear"
				checked={$data.method === "linear"}
			/>
			<label for="scales">Linear</label>
		</div>

		<div>
			<input
				type="checkbox"
				id="exponential"
				name="exponential"
				checked={$data.method === "exponential"}
			/>
			<label for="scales">Exponential</label>
		</div>

		<div>
			<input
				type="checkbox"
				id="log"
				name="log"
				checked={$data.method === "log"}
			/>
			<label for="scales">Log</label>
		</div>

		<div>
			<input
				type="checkbox"
				id="sCurve"
				name="sCurve"
				checked={$data.method === "sCurve"}
			/>
			<label for="scales">S-Curve</label>
		</div>
	</fieldset>

	{#if showButtons}
		<div class="col">
			<h3>DATA</h3>
			<div class="data">
				<p>Method --- {$data.method}</p>
				<p>Playing --- {$data.playing}</p>
				<p>Duration --- {$data.duration}</p>
				<p>Current Time --- {$data.currentTime}</p>
				<p>Gain --- {$data.gain}</p>
			</div>
		</div>
	{/if}
</div>

{#if showButtons}
	<div class="row">
		<button on:click={$data.playing ? pause : play}>
			{#if $data.playing}
				⏸️ Pause
			{:else}
				▶️ Play
			{/if}
		</button>
	</div>
{/if}

<style>
	h1 {
		text-align: center;
		padding: 2rem;
	}
	.row {
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		padding: 2rem;
		gap: 1rem;
	}

	.col {
		display: flex;
		flex-direction: column;
	}

	.data {
		width: 30rem;
	}

	p {
		margin: 0;
	}
	button {
		margin: 1rem;
		padding: 1rem;
		font-size: 1.5rem;
		border-radius: 1rem;
		background: rgb(186, 184, 184);
	}
</style>
