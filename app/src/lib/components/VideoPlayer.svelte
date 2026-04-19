<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	type Props = {
		src: string;
	};

	let { src }: Props = $props();

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null = null;
	let decoder: VideoDecoder | null = null;
	let videoWidth = $state(0);
	let videoHeight = $state(0);
	let currentFrame = $state(0);
	let totalFrames = $state(0);
	let isPlaying = $state(false);
	let isLoaded = $state(false);
	let error = $state('');

	let frames: EncodedVideoChunk[] = [];
	let pendingFrame: VideoFrame | null = null;
	let animationId: number | null = null;
	let frameRate = $state(29.97);
	let startTime = 0;
	let decoderConfig: VideoDecoderConfig | null = null;
	let blobUrl: string | null = null;

	function initDecoder() {
		decoder = new VideoDecoder({
			output: (frame: VideoFrame) => {
				if (pendingFrame) {
					pendingFrame.close();
				}
				pendingFrame = frame;
			},
			error: (e: DOMException) => {
				error = `Decoder error: ${e.message}`;
				console.error('VideoDecoder error:', e);
			}
		});
	}

	async function loadVideo() {
		try {
			error = '';
			const response = await fetch(src);
			if (!response.ok) throw new Error(`Failed to fetch video: ${response.status}`);

			const buffer = await response.arrayBuffer();
			await demuxAndDecode(buffer);
		} catch (e: any) {
			error = e.message;
			console.error('Failed to load video:', e);
		}
	}

	async function demuxAndDecode(buffer: ArrayBuffer) {
		// Use mp4box.js-style manual parsing or basic approach
		// For now, use a MediaSource + video element to extract frames via WebCodecs
		const blob = new Blob([buffer], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);
		blobUrl = url; // Keep the URL alive for the lifetime of the component

		const videoEl = document.createElement('video');
		videoEl.muted = true;
		videoEl.playsInline = true;
		videoEl.preload = 'auto';
		videoEl.src = url;

		await new Promise<void>((resolve, reject) => {
			videoEl.onloadedmetadata = () => {
				videoWidth = videoEl.videoWidth;
				videoHeight = videoEl.videoHeight;
				frameRate = 29.97; // Default NTSC
				totalFrames = Math.floor(videoEl.duration * frameRate);

				if (canvas) {
					canvas.width = videoWidth;
					canvas.height = videoHeight;
					ctx = canvas.getContext('2d');
				}
				resolve();
			};
			videoEl.onerror = () => reject(new Error('Failed to load video metadata'));
		});

		// Use the video element + canvas for initial frame extraction
		// This provides a working baseline before implementing raw WebCodecs demuxing
		await extractFramesViaCanvas(videoEl);
	}

	async function extractFramesViaCanvas(videoEl: HTMLVideoElement) {
		// Add video element to document (offscreen, for playback support)
		videoEl.style.display = 'none';
		videoEl.crossOrigin = 'anonymous';
		document.body.appendChild(videoEl);

		isLoaded = true;

		// Draw the first frame
		videoEl.currentTime = 0;
		await new Promise<void>((resolve) => {
			const onSeeked = () => {
				try {
					drawVideoFrame(videoEl);
				} catch (e) {
					console.error('Failed to draw initial frame:', e);
				}
				videoEl.removeEventListener('seeked', onSeeked);
				resolve();
			};
			videoEl.addEventListener('seeked', onSeeked);
		});

		// Store video element for playback
		(window as any).__videoEl = videoEl;
		(window as any).__videoContainer = videoEl;
	}

	function drawVideoFrame(source: HTMLVideoElement | VideoFrame) {
		if (!ctx || !canvas) return;
		try {
			ctx.drawImage(source as any, 0, 0, canvas.width, canvas.height);
		} catch (e) {
			console.error('Failed to draw frame:', e);
		}
	}

	function play() {
		if (!isLoaded) return;
		const videoEl = (window as any).__videoEl as HTMLVideoElement;
		if (!videoEl) {
			error = 'Video element not found';
			return;
		}

		isPlaying = true;
		videoEl.play().catch((err: any) => {
			error = `Playback error: ${err.message}`;
			isPlaying = false;
		});

		const renderLoop = () => {
			if (!isPlaying) return;

			try {
				drawVideoFrame(videoEl);
				currentFrame = Math.floor(videoEl.currentTime * frameRate);
			} catch (e) {
				console.error('Render error:', e);
			}

			if (videoEl.ended) {
				isPlaying = false;
				return;
			}
			animationId = requestAnimationFrame(renderLoop);
		};
		animationId = requestAnimationFrame(renderLoop);
	}

	function pause() {
		isPlaying = false;
		if (browser) {
			const videoEl = (window as any).__videoEl as HTMLVideoElement;
			if (videoEl) {
				videoEl.pause();
			}
		}
		if (animationId !== null) {
			cancelAnimationFrame(animationId);
			animationId = null;
		}
	}

	function seek(frameNumber: number) {
		const videoEl = (window as any).__videoEl as HTMLVideoElement;
		if (!videoEl) return;

		const wasPlaying = isPlaying;
		if (wasPlaying) {
			videoEl.pause();
		}

		currentFrame = frameNumber;
		videoEl.currentTime = frameNumber / frameRate;

		const onSeeked = () => {
			drawVideoFrame(videoEl);
			if (wasPlaying) {
				videoEl.play().catch((err: any) => {
					console.error('Failed to resume playback after seek:', err);
				});
			}
			videoEl.removeEventListener('seeked', onSeeked);
		};
		videoEl.addEventListener('seeked', onSeeked);
	}

	function handleSliderInput(e: Event) {
		const target = e.target as HTMLInputElement;
		seek(parseInt(target.value));
	}

	function stepForward() {
		if (currentFrame < totalFrames - 1) {
			seek(currentFrame + 1);
		}
	}

	function stepBackward() {
		if (currentFrame > 0) {
			seek(currentFrame - 1);
		}
	}

	function formatTimecode(frame: number): string {
		const totalSeconds = frame / frameRate;
		const hours = Math.floor(totalSeconds / 3600);
		const minutes = Math.floor((totalSeconds % 3600) / 60);
		const seconds = Math.floor(totalSeconds % 60);
		const remainingFrames = Math.floor(frame % frameRate);
		return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}:${String(remainingFrames).padStart(2, '0')}`;
	}

	onMount(() => {
		if (!('VideoDecoder' in window)) {
			error = 'WebCodecs API is not supported in this browser.';
			return;
		}
		loadVideo();
	});

	onDestroy(() => {
		if (!browser) return;

		pause();

		if (pendingFrame) {
			pendingFrame.close();
			pendingFrame = null;
		}

		if (decoder && decoder.state !== 'closed') {
			decoder.close();
		}

		const videoEl = (window as any).__videoEl;
		if (videoEl) {
			videoEl.pause();
			videoEl.src = '';
			if (videoEl.parentNode) {
				videoEl.parentNode.removeChild(videoEl);
			}
			delete (window as any).__videoEl;
			delete (window as any).__videoContainer;
		}

		if (blobUrl) {
			URL.revokeObjectURL(blobUrl);
			blobUrl = null;
		}
	});
</script>

<div class="flex flex-col gap-4">
	{#if error}
		<div class="bg-red-900/50 border border-red-700 text-red-200 px-4 py-3 rounded-lg text-sm">
			{error}
		</div>
	{/if}

	<!-- Canvas Viewport -->
	<div class="relative bg-black rounded-lg overflow-hidden border border-slate-700 max-h-96">
		<canvas
			bind:this={canvas}
			class="block w-full h-auto max-h-96 object-contain"
		></canvas>

		{#if !isLoaded && !error}
			<div class="absolute inset-0 flex items-center justify-center bg-black/80">
				<p class="text-slate-400 text-sm">Loading video...</p>
			</div>
		{/if}
	</div>

	<!-- Timecode Display -->
	<div class="flex items-center justify-between text-xs font-mono text-slate-400">
		<span>{formatTimecode(currentFrame)}</span>
		<span>Frame {currentFrame} / {totalFrames}</span>
		<span>{videoWidth}×{videoHeight} @ {frameRate.toFixed(2)}fps</span>
	</div>

	<!-- Timeline Scrubber -->
	<div class="w-full">
		<input
			type="range"
			min="0"
			max={totalFrames - 1}
			value={currentFrame}
			oninput={handleSliderInput}
			disabled={!isLoaded}
			class="w-full h-2 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500 disabled:opacity-50 disabled:cursor-not-allowed"
		/>
	</div>

	<!-- Transport Controls -->
	<div class="flex items-center justify-between w-full">
		<div></div>

		<div class="flex items-center gap-3">
			<button
				onclick={stepBackward}
				disabled={!isLoaded}
				class="px-3 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 hover:text-white transition-colors disabled:opacity-40 disabled:cursor-not-allowed"
			>
				◀◀
			</button>

			{#if isPlaying}
				<button
					onclick={pause}
					class="px-5 py-2 bg-blue-600 border border-blue-500 rounded-md text-white text-sm font-medium hover:bg-blue-500 transition-colors"
				>
					⏸ Pause
				</button>
			{:else}
				<button
					onclick={play}
					disabled={!isLoaded}
					class="px-5 py-2 bg-blue-600 border border-blue-500 rounded-md text-white text-sm font-medium hover:bg-blue-500 transition-colors disabled:opacity-40 disabled:cursor-not-allowed"
				>
					▶ Play
				</button>
			{/if}

			<button
				onclick={stepForward}
				disabled={!isLoaded}
				class="px-3 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 hover:text-white transition-colors disabled:opacity-40 disabled:cursor-not-allowed"
			>
				▶▶
			</button>
		</div>

		<div class="flex items-center gap-3">
			<div class="w-px h-6 bg-slate-700"></div>

			<button
				disabled={!isLoaded}
				class="px-3 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 hover:text-white transition-colors disabled:opacity-40 disabled:cursor-not-allowed"
				title="Trim"
			>
				✂
			</button>
		</div>
	</div>
</div>
