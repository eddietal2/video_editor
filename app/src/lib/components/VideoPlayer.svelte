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
	let isTrimMode = $state(false);
	let trimStartFrame = $state<number | null>(null);
	let trimEndFrame = $state<number | null>(null);
	let hoverFrame = $state<number | null>(null);
	let draggingTrimPoint = $state<'start' | 'end' | null>(null);
	let trimDisplayMode = $state<'frames' | 'seconds'>('frames');
	let showTrimConfirmModal = $state(false);
	let pendingTrimData = $state<{start: number, end: number, duration: string} | null>(null);

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
		const frame = parseInt(target.value);
		
		if (isTrimMode) {
			// In trim mode, set start or end point
			if (trimStartFrame === null) {
				trimStartFrame = frame;
			} else if (trimEndFrame === null) {
				trimEndFrame = frame;
			}
		}
		
		seek(frame);
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

	function formatTrimRangeDisplay(): string {
		if (trimStartFrame === null || trimEndFrame === null) return '';

		const start = Math.min(trimStartFrame, trimEndFrame);
		const end = Math.max(trimStartFrame, trimEndFrame);

		if (trimDisplayMode === 'frames') {
			return `Frame ${start} → ${end} (${end - start + 1} frames)`;
		} else {
			const startSeconds = (start / frameRate).toFixed(2);
			const endSeconds = (end / frameRate).toFixed(2);
			const durationSeconds = ((end - start + 1) / frameRate).toFixed(2);
			return `${startSeconds}s → ${endSeconds}s (${durationSeconds}s duration)`;
		}
	}

	function toggleTrimDisplayMode() {
		trimDisplayMode = trimDisplayMode === 'frames' ? 'seconds' : 'frames';
	}

	function executeTrim() {
		if (trimStartFrame === null || trimEndFrame === null) return;
		
		const start = Math.min(trimStartFrame, trimEndFrame);
		const end = Math.max(trimStartFrame, trimEndFrame);
		const durationSeconds = ((end - start + 1) / frameRate).toFixed(2);
		
		// Store trim data and show modal
		pendingTrimData = {
			start,
			end,
			duration: durationSeconds
		};
		showTrimConfirmModal = true;
	}

	function confirmTrim() {
		if (!pendingTrimData) return;
		
		const { start, end, duration } = pendingTrimData;
		
		// Log the trim action
		console.log(`Trim executed: Frame ${start} to ${end}`);
		console.log(`Duration: ${duration}s`);
		
		// TODO: Implement actual trim logic here
		// This could involve:
		// - Creating a new video file with only the trimmed segment
		// - Updating the video source
		// - Exporting the trimmed video
		
		// Close modal and reset
		showTrimConfirmModal = false;
		pendingTrimData = null;
		error = '';
	}

	function dismissTrimModal() {
		showTrimConfirmModal = false;
		pendingTrimData = null;
	}

	function toggleTrimMode() {
		isTrimMode = !isTrimMode;
	}

	function cancelTrimMode() {
		isTrimMode = false;
		trimStartFrame = null;
		trimEndFrame = null;
		hoverFrame = null;
		draggingTrimPoint = null;
		trimDisplayMode = 'frames';
	}

	function handleTimelineMouseMove(e: MouseEvent) {
		if (!isTrimMode) return;
		
		const slider = e.currentTarget as HTMLInputElement;
		const rect = slider.getBoundingClientRect();
		const percent = (e.clientX - rect.left) / rect.width;
		const frame = Math.max(0, Math.min(totalFrames - 1, Math.round(percent * (totalFrames - 1))));
		
		if (draggingTrimPoint === 'start') {
			trimStartFrame = frame;
		} else if (draggingTrimPoint === 'end') {
			trimEndFrame = frame;
		} else {
			hoverFrame = frame;
		}
	}

	function handleTimelineMouseLeave() {
		if (!draggingTrimPoint) {
			hoverFrame = null;
		}
	}

	function handleMouseDown(point: 'start' | 'end') {
		draggingTrimPoint = point;
	}

	function handleMouseUp() {
		draggingTrimPoint = null;
	}

	onMount(() => {
		if (!('VideoDecoder' in window)) {
			error = 'WebCodecs API is not supported in this browser.';
			return;
		}
		loadVideo();

		// Add global event listeners for drag support
		if (browser) {
			const handleGlobalMouseMove = (e: MouseEvent) => {
				if (draggingTrimPoint && isTrimMode) {
					// Find the timeline slider and calculate position
					const slider = document.querySelector('input[type="range"]') as HTMLInputElement;
					if (slider) {
						const rect = slider.getBoundingClientRect();
						const percent = (e.clientX - rect.left) / rect.width;
						const frame = Math.max(0, Math.min(totalFrames - 1, Math.round(percent * (totalFrames - 1))));
						
						if (draggingTrimPoint === 'start') {
							trimStartFrame = frame;
						} else if (draggingTrimPoint === 'end') {
							trimEndFrame = frame;
						}
					}
				}
			};

			document.addEventListener('mousemove', handleGlobalMouseMove);
			document.addEventListener('mouseup', handleMouseUp);

			return () => {
				document.removeEventListener('mousemove', handleGlobalMouseMove);
				document.removeEventListener('mouseup', handleMouseUp);
			};
		}
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
	<div 
		class={`w-full p-2 rounded-lg transition-colors ${isTrimMode ? 'bg-blue-950 border border-blue-600' : ''}`}
		onmousemove={handleTimelineMouseMove}
		onmouseleave={handleTimelineMouseLeave}
	>
		{#if isTrimMode}
			<div class="flex items-center justify-between mb-2">
				<div class="text-xs text-slate-400">
					{#if trimStartFrame === null}
						<span class="text-yellow-500">Click to set Start Point</span>
					{:else if trimEndFrame === null}
						<span class="text-yellow-500">Click to set End Point</span>
					{:else}
						<span class="text-green-500">{formatTrimRangeDisplay()}</span>
					{/if}
				</div>
				
				{#if trimStartFrame !== null && trimEndFrame !== null}
					<div class="flex items-center gap-2">
						<button
							onclick={toggleTrimDisplayMode}
							class="px-2 py-1 bg-slate-700 border border-slate-600 rounded text-slate-300 text-xs hover:bg-slate-600 transition-colors"
							title="Toggle between frames and seconds"
						>
							{trimDisplayMode === 'frames' ? '🎬 Frames' : '⏱ Seconds'}
						</button>
						
						<button
							onclick={executeTrim}
							class="px-3 py-1 bg-green-600 border border-green-500 rounded text-white text-xs font-medium hover:bg-green-500 transition-colors"
							title="Apply Trim"
						>
							✓ Apply Trim
						</button>
					</div>
				{/if}
			</div>
		{/if}
		
		<div class="relative">
			{#if trimStartFrame !== null && trimEndFrame !== null}
				<div
					class="absolute top-0 h-full bg-yellow-500/30 border-l-2 border-r-2 border-yellow-500 rounded pointer-events-none"
					style={`left: ${(Math.min(trimStartFrame, trimEndFrame) / (totalFrames - 1)) * 100}%; right: ${100 - (Math.max(trimStartFrame, trimEndFrame) / (totalFrames - 1)) * 100}%; z-index: 1;`}
				></div>
			{/if}
			
			{#if isTrimMode && hoverFrame !== null && trimStartFrame === null}
				<div
					class="absolute top-1/2 -translate-y-1/2 w-0.5 h-4 bg-yellow-400 pointer-events-none"
					style={`left: ${(hoverFrame / (totalFrames - 1)) * 100}%; z-index: 3;`}
				></div>
			{/if}

			{#if isTrimMode && hoverFrame !== null && trimStartFrame !== null && trimEndFrame === null}
				<div
					class="absolute top-1/2 -translate-y-1/2 w-0.5 h-4 bg-yellow-400 pointer-events-none"
					style={`left: ${(hoverFrame / (totalFrames - 1)) * 100}%; z-index: 3;`}
				></div>
			{/if}

			{#if trimStartFrame !== null}
				<div
					class="absolute top-1/2 -translate-y-1/2 w-1 h-6 bg-yellow-400 border border-yellow-300 rounded cursor-grab active:cursor-grabbing hover:bg-yellow-300"
					style={`left: ${(trimStartFrame / (totalFrames - 1)) * 100}%; transform: translateY(-50%) translateX(-50%); z-index: 4;`}
					onmousedown={() => handleMouseDown('start')}
					title="Drag to adjust start point"
				></div>
			{/if}

			{#if trimEndFrame !== null}
				<div
					class="absolute top-1/2 -translate-y-1/2 w-1 h-6 bg-yellow-400 border border-yellow-300 rounded cursor-grab active:cursor-grabbing hover:bg-yellow-300"
					style={`left: ${(trimEndFrame / (totalFrames - 1)) * 100}%; transform: translateY(-50%) translateX(-50%); z-index: 4;`}
					onmousedown={() => handleMouseDown('end')}
					title="Drag to adjust end point"
				></div>
			{/if}
			
			<input
				type="range"
				min="0"
				max={totalFrames - 1}
				value={currentFrame}
				oninput={handleSliderInput}
				disabled={!isLoaded}
				class="relative w-full h-2 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500 disabled:opacity-50 disabled:cursor-not-allowed"
				style="z-index: 2;"
			/>
		</div>
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
				onclick={toggleTrimMode}
				class={`px-3 py-2 border rounded-md text-sm font-medium transition-colors ${isTrimMode ? 'bg-blue-600 border-blue-500 text-white hover:bg-blue-500' : 'bg-slate-800 border-slate-600 text-slate-300 hover:bg-slate-700 hover:text-white'} disabled:opacity-40 disabled:cursor-not-allowed`}
				title="Trim Mode"
			>
				✂ Trim Mode
			</button>

			{#if isTrimMode}
				<button
					onclick={cancelTrimMode}
					class="px-3 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 hover:text-white transition-colors"
					title="Cancel Trim"
				>
					✕
				</button>
			{/if}
		</div>
	</div>
</div>

<!-- Trim Confirmation Modal -->
{#if showTrimConfirmModal && pendingTrimData}
	<div class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
		<div class="bg-slate-900 border border-slate-700 rounded-lg shadow-2xl max-w-sm mx-4">
			<!-- Header -->
			<div class="border-b border-slate-700 px-6 py-4">
				<h2 class="text-lg font-bold text-white">Confirm Trim</h2>
			</div>
			
			<!-- Content -->
			<div class="px-6 py-6 space-y-4">
				<div class="bg-slate-800/50 rounded-lg p-4 space-y-3">
					<div class="flex justify-between items-center">
						<span class="text-slate-400 text-sm">Start Frame:</span>
						<span class="text-white font-mono text-sm">{pendingTrimData.start}</span>
					</div>
					<div class="flex justify-between items-center">
						<span class="text-slate-400 text-sm">End Frame:</span>
						<span class="text-white font-mono text-sm">{pendingTrimData.end}</span>
					</div>
					<div class="border-t border-slate-700 pt-3 flex justify-between items-center">
						<span class="text-slate-300 text-sm font-medium">Duration:</span>
						<span class="text-green-400 font-mono text-sm font-bold">{pendingTrimData.duration}s</span>
					</div>
				</div>
				
				<p class="text-slate-400 text-xs">This action will process the trimmed video segment.</p>
			</div>
			
			<!-- Footer -->
			<div class="border-t border-slate-700 px-6 py-4 flex items-center gap-3 justify-end">
				<button
					onclick={dismissTrimModal}
					class="px-4 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 transition-colors"
				>
					Cancel
				</button>
				
				<button
					onclick={confirmTrim}
					class="px-4 py-2 bg-green-600 border border-green-500 rounded-md text-white text-sm font-medium hover:bg-green-500 transition-colors"
				>
					Apply Trim
				</button>
			</div>
		</div>
	</div>
{/if}
