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
	let isTrimmingVideo = $state(false);
	let trimProgress = $state(0);

	// Segment-based editing: track which portions of the original video are kept
	let segments = $state<{startFrame: number, endFrame: number}[]>([]);
	let originalTotalFrames = 0;

	// Filmstrip thumbnails
	let thumbnails = $state<string[]>([]);
	let thumbnailCount = 60;
	let trackContainer: HTMLDivElement;
	let trackHoverFrame = $state<number | null>(null);

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
				originalTotalFrames = Math.floor(videoEl.duration * frameRate);
				totalFrames = originalTotalFrames;
				segments = [{ startFrame: 0, endFrame: originalTotalFrames - 1 }];

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

		// Create a second video element for double-buffered segment playback
		const videoB = document.createElement('video');
		videoB.muted = true;
		videoB.playsInline = true;
		videoB.preload = 'auto';
		videoB.src = videoEl.src;
		videoB.style.display = 'none';
		videoB.crossOrigin = 'anonymous';
		document.body.appendChild(videoB);

		// Wait for videoB to be ready
		await new Promise<void>((resolve) => {
			videoB.onloadeddata = () => resolve();
			if (videoB.readyState >= 2) resolve();
		});

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

		// Store video elements for playback
		(window as any).__videoEl = videoEl;
		(window as any).__videoElB = videoB;
		(window as any).__videoContainer = videoEl;

		// Generate filmstrip thumbnails
		await generateThumbnails(videoEl);
	}

	async function generateThumbnails(videoEl: HTMLVideoElement) {
		const thumbCanvas = document.createElement('canvas');
		const thumbW = 160;
		const thumbH = Math.round(thumbW * (videoHeight / videoWidth));
		thumbCanvas.width = thumbW;
		thumbCanvas.height = thumbH;
		const thumbCtx = thumbCanvas.getContext('2d')!;

		const duration = videoEl.duration;
		const count = Math.min(thumbnailCount, originalTotalFrames);
		const interval = duration / count;
		const thumbs: string[] = [];

		for (let i = 0; i < count; i++) {
			const time = i * interval;
			await new Promise<void>((resolve) => {
				videoEl.currentTime = time;
				const onSeeked = () => {
					thumbCtx.drawImage(videoEl, 0, 0, thumbW, thumbH);
					thumbs.push(thumbCanvas.toDataURL('image/jpeg', 0.5));
					videoEl.removeEventListener('seeked', onSeeked);
					resolve();
				};
				videoEl.addEventListener('seeked', onSeeked);
			});
		}

		thumbnails = thumbs;

		// Reset to frame 0
		videoEl.currentTime = 0;
		await new Promise<void>((resolve) => {
			const onSeeked = () => {
				drawVideoFrame(videoEl);
				videoEl.removeEventListener('seeked', onSeeked);
				resolve();
			};
			videoEl.addEventListener('seeked', onSeeked);
		});
	}

	function drawVideoFrame(source: HTMLVideoElement | VideoFrame) {
		if (!ctx || !canvas) return;
		try {
			ctx.drawImage(source as any, 0, 0, canvas.width, canvas.height);
		} catch (e) {
			console.error('Failed to draw frame:', e);
		}
	}

	// Convert a virtual (timeline) frame to the real frame in the original video
	function virtualToRealFrame(virtualFrame: number): number {
		let accumulated = 0;
		for (const seg of segments) {
			const segLen = seg.endFrame - seg.startFrame + 1;
			if (virtualFrame < accumulated + segLen) {
				return seg.startFrame + (virtualFrame - accumulated);
			}
			accumulated += segLen;
		}
		return segments[segments.length - 1].endFrame;
	}

	// Convert a real frame to virtual (timeline) frame, or -1 if not in any segment
	function realToVirtualFrame(realFrame: number): number {
		let accumulated = 0;
		for (const seg of segments) {
			if (realFrame >= seg.startFrame && realFrame <= seg.endFrame) {
				return accumulated + (realFrame - seg.startFrame);
			}
			accumulated += seg.endFrame - seg.startFrame + 1;
		}
		return -1;
	}

	// Find which segment a real frame belongs to, and the next segment
	function findSegmentForRealFrame(realFrame: number): { segIndex: number, isInSegment: boolean } {
		for (let i = 0; i < segments.length; i++) {
			if (realFrame >= segments[i].startFrame && realFrame <= segments[i].endFrame) {
				return { segIndex: i, isInSegment: true };
			}
		}
		// Find the next segment after this real frame
		for (let i = 0; i < segments.length; i++) {
			if (segments[i].startFrame > realFrame) {
				return { segIndex: i, isInSegment: false };
			}
		}
		return { segIndex: -1, isInSegment: false };
	}

	function getVirtualTotalFrames(): number {
		return segments.reduce((sum, seg) => sum + (seg.endFrame - seg.startFrame + 1), 0);
	}

	// Find which segment index a real frame is inside
	function getSegmentIndex(realFrame: number): number {
		for (let i = 0; i < segments.length; i++) {
			if (realFrame >= segments[i].startFrame && realFrame <= segments[i].endFrame) {
				return i;
			}
		}
		return -1;
	}

	// Pre-seek the standby video to the next segment's start frame
	function preSeekStandby(standbyEl: HTMLVideoElement, currentSegIdx: number) {
		const nextIdx = currentSegIdx + 1;
		if (nextIdx < segments.length) {
			standbyEl.currentTime = segments[nextIdx].startFrame / frameRate;
		}
	}

	function play() {
		if (!isLoaded) return;
		const videoA = (window as any).__videoEl as HTMLVideoElement;
		const videoB = (window as any).__videoElB as HTMLVideoElement;
		if (!videoA || !videoB) {
			error = 'Video elements not found';
			return;
		}

		isPlaying = true;

		// Determine which segment we're currently in
		const realFrame = virtualToRealFrame(currentFrame);
		let currentSegIdx = getSegmentIndex(realFrame);
		if (currentSegIdx < 0) currentSegIdx = 0;

		// Set up active/standby
		let activeEl = videoA;
		let standbyEl = videoB;

		// Ensure active is at correct position and playing
		activeEl.currentTime = realFrame / frameRate;
		activeEl.play().catch((err: any) => {
			error = `Playback error: ${err.message}`;
			isPlaying = false;
		});

		// Pre-seek standby to the next segment
		preSeekStandby(standbyEl, currentSegIdx);

		const renderLoop = () => {
			if (!isPlaying) return;

			try {
				const activeRealFrame = Math.floor(activeEl.currentTime * frameRate);

				// Check if we've passed the end of the current segment
				if (currentSegIdx >= 0 && currentSegIdx < segments.length &&
					activeRealFrame > segments[currentSegIdx].endFrame) {

					const nextIdx = currentSegIdx + 1;
					if (nextIdx < segments.length) {
						// Swap: standby becomes active (it's already seeked to the right spot)
						activeEl.pause();
						const temp = activeEl;
						activeEl = standbyEl;
						standbyEl = temp;

						currentSegIdx = nextIdx;
						activeEl.play().catch(() => {});

						// Pre-seek the new standby to the NEXT segment
						preSeekStandby(standbyEl, currentSegIdx);
					} else {
						// No more segments
						isPlaying = false;
						activeEl.pause();
						return;
					}
				}

				// Always draw from the active element
				drawVideoFrame(activeEl);
				const virtualFrame = realToVirtualFrame(Math.floor(activeEl.currentTime * frameRate));
				if (virtualFrame >= 0) {
					currentFrame = virtualFrame;
				}
			} catch (e) {
				console.error('Render error:', e);
			}

			if (activeEl.ended || currentFrame >= totalFrames - 1) {
				isPlaying = false;
				activeEl.pause();
				return;
			}
			animationId = requestAnimationFrame(renderLoop);
		};
		animationId = requestAnimationFrame(renderLoop);
	}

	function pause() {
		isPlaying = false;
		if (browser) {
			const videoA = (window as any).__videoEl as HTMLVideoElement;
			const videoB = (window as any).__videoElB as HTMLVideoElement;
			if (videoA) videoA.pause();
			if (videoB) videoB.pause();
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

		// Map virtual frame to real frame in the original video
		const realFrame = virtualToRealFrame(frameNumber);
		currentFrame = frameNumber;
		videoEl.currentTime = realFrame / frameRate;

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
		if (!pendingTrimData || !browser) return;
		processTrim(pendingTrimData.start, pendingTrimData.end);
	}

	function processTrim(trimStart: number, trimEnd: number) {
		if (!browser) return;

		const videoEl = (window as any).__videoEl as HTMLVideoElement;
		if (!videoEl) return;

		// Convert virtual trim range to real frames
		const realStart = virtualToRealFrame(trimStart);
		const realEnd = virtualToRealFrame(trimEnd);

		// Remove the trimmed range from the segment list
		const newSegments: {startFrame: number, endFrame: number}[] = [];

		for (const seg of segments) {
			// Segment is entirely before the trim — keep it
			if (seg.endFrame < realStart) {
				newSegments.push({ ...seg });
				continue;
			}

			// Segment is entirely after the trim — keep it
			if (seg.startFrame > realEnd) {
				newSegments.push({ ...seg });
				continue;
			}

			// Segment overlaps with trim — split it
			// Part before the trim
			if (seg.startFrame < realStart) {
				newSegments.push({ startFrame: seg.startFrame, endFrame: realStart - 1 });
			}

			// Part after the trim
			if (seg.endFrame > realEnd) {
				newSegments.push({ startFrame: realEnd + 1, endFrame: seg.endFrame });
			}
		}

		segments = newSegments;
		totalFrames = getVirtualTotalFrames();

		// Clamp current frame
		if (currentFrame >= totalFrames) {
			currentFrame = Math.max(0, totalFrames - 1);
		}

		// Reset trim state
		showTrimConfirmModal = false;
		pendingTrimData = null;
		trimStartFrame = null;
		trimEndFrame = null;
		hoverFrame = null;
		draggingTrimPoint = null;
		trimDisplayMode = 'frames';
		isTrimMode = false;

		// Seek to current position (now remapped)
		seek(Math.min(currentFrame, totalFrames - 1));
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

	async function exportVideo() {
		if (!browser || !isLoaded || !canvas) {
			error = 'Cannot export: video not loaded';
			return;
		}

		try {
			error = '';
			const videoEl = (window as any).__videoEl as HTMLVideoElement;
			if (!videoEl) {
				error = 'Video element not found';
				return;
			}

			// Get canvas stream for encoding
			const stream = canvas.captureStream(frameRate) as MediaStream;
			const recorder = new MediaRecorder(stream, {
				mimeType: 'video/mp4',
				videoBitsPerSecond: 5000000 // 5 Mbps
			});

			const chunks: Blob[] = [];
			recorder.ondataavailable = (e: BlobEvent) => {
				if (e.data.size > 0) {
					chunks.push(e.data);
				}
			};

			recorder.onerror = (e: Event) => {
				const errorEvent = e as Event & { error?: DOMException };
				error = `Recording error: ${errorEvent.error?.message || 'Unknown error'}`;
				console.error('MediaRecorder error:', errorEvent.error);
			};

			// Wait a bit for the stream to be ready
			await new Promise(resolve => setTimeout(resolve, 100));

			recorder.start();
			const wasPlaying = isPlaying;
			if (wasPlaying) {
				pause();
			}

			// Render all kept frames
			const framesToRender = getVirtualTotalFrames();
			for (let i = 0; i < framesToRender; i++) {
				const realFrame = virtualToRealFrame(i);
				if (realFrame >= 0) {
					videoEl.currentTime = realFrame / frameRate;
					await new Promise(resolve => {
						const onSeeked = () => {
							drawVideoFrame(videoEl);
							videoEl.removeEventListener('seeked', onSeeked);
							// Give the renderer time to draw
							requestAnimationFrame(() => resolve(null));
						};
						videoEl.addEventListener('seeked', onSeeked);
					});
				}
			}

			// Stop recording
			recorder.stop();

			// Wait for all chunks to be collected
			await new Promise(resolve => {
				recorder.onstop = () => resolve(null);
			});

			// Create blob and download
			const blob = new Blob(chunks, { type: 'video/mp4' });
			const url = URL.createObjectURL(blob);
			const a = document.createElement('a');
			a.href = url;
			a.download = `video_${Date.now()}.mp4`;
			document.body.appendChild(a);
			a.click();
			document.body.removeChild(a);
			URL.revokeObjectURL(url);

			// Restore playback state
			currentFrame = 0;
			seek(0);
			if (wasPlaying) {
				play();
			}
		} catch (e: any) {
			error = `Export failed: ${e.message}`;
			console.error('Export error:', e);
		}
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

	function handleTrackClick(e: MouseEvent) {
		if (!trackContainer || !isLoaded) return;
		const rect = trackContainer.getBoundingClientRect();
		const percent = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
		const frame = Math.round(percent * (totalFrames - 1));

		if (isTrimMode) {
			if (trimStartFrame === null) {
				trimStartFrame = frame;
			} else if (trimEndFrame === null) {
				trimEndFrame = frame;
			}
		}

		seek(frame);
	}

	function handleTrackMouseMove(e: MouseEvent) {
		if (!trackContainer) return;
		const rect = trackContainer.getBoundingClientRect();
		const percent = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
		const frame = Math.round(percent * (totalFrames - 1));
		trackHoverFrame = frame;

		if (isTrimMode) {
			if (draggingTrimPoint === 'start') {
				trimStartFrame = frame;
			} else if (draggingTrimPoint === 'end') {
				trimEndFrame = frame;
			} else {
				hoverFrame = frame;
			}
		}
	}

	function handleTrackMouseLeave() {
		trackHoverFrame = null;
		if (!draggingTrimPoint) {
			hoverFrame = null;
		}
	}

	// Get thumbnail index for a given virtual frame position within total frames
	function getThumbnailsForSegments(): {src: string, widthPercent: number}[] {
		if (thumbnails.length === 0) return [];
		const result: {src: string, widthPercent: number}[] = [];
		const virtualTotal = getVirtualTotalFrames();
		if (virtualTotal === 0) return [];

		for (const seg of segments) {
			const segLen = seg.endFrame - seg.startFrame + 1;
			const widthPercent = (segLen / virtualTotal) * 100;
			// Pick thumbnails from this segment's real frame range
			const framesPerThumb = originalTotalFrames / thumbnails.length;
			const segThumbStart = Math.floor(seg.startFrame / framesPerThumb);
			const segThumbEnd = Math.ceil(seg.endFrame / framesPerThumb);
			const segThumbs = thumbnails.slice(segThumbStart, Math.max(segThumbStart + 1, segThumbEnd));

			for (let i = 0; i < segThumbs.length; i++) {
				result.push({
					src: segThumbs[i],
					widthPercent: widthPercent / segThumbs.length
				});
			}
		}
		return result;
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
				if (draggingTrimPoint && isTrimMode && trackContainer) {
					const rect = trackContainer.getBoundingClientRect();
					const percent = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
					const frame = Math.round(percent * (totalFrames - 1));
					
					if (draggingTrimPoint === 'start') {
						trimStartFrame = frame;
					} else if (draggingTrimPoint === 'end') {
						trimEndFrame = frame;
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

		const videoElB = (window as any).__videoElB;
		if (videoElB) {
			videoElB.pause();
			videoElB.src = '';
			if (videoElB.parentNode) {
				videoElB.parentNode.removeChild(videoElB);
			}
			delete (window as any).__videoElB;
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

	<!-- Trim Mode Controls -->
	{#if isTrimMode}
		<div class="flex items-center justify-between px-2">
			<div class="text-xs text-slate-400">
				{#if trimStartFrame === null}
					<span class="text-yellow-500">Click track to set Start Point</span>
				{:else if trimEndFrame === null}
					<span class="text-yellow-500">Click track to set End Point</span>
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

	<!-- Export Button (always visible) -->
	<div class="flex justify-end px-2">
		<button
			onclick={exportVideo}
			disabled={!isLoaded}
			class="px-4 py-2 bg-blue-600 border border-blue-500 rounded-md text-white text-sm font-medium hover:bg-blue-500 transition-colors disabled:opacity-40 disabled:cursor-not-allowed flex items-center gap-2"
			title="Export video with current edits as MP4"
		>
			↓ Export MP4
		</button>
	</div>

	<!-- Video Track (Filmstrip) -->
	<div class="w-full">
		<!-- Time ruler -->
		<div class="relative h-5 mb-1 select-none flex items-center">
			{#if totalFrames > 0}
				{@const totalSec = totalFrames / frameRate}
				{@const tickInterval = totalSec <= 10 ? 1 : totalSec <= 30 ? 2 : totalSec <= 60 ? 5 : 10}
				{@const tickCount = Math.floor(totalSec / tickInterval)}
				<div class="relative flex-1">
					{#each Array(tickCount + 1) as _, i}
						{@const sec = i * tickInterval}
						{@const pct = (sec / totalSec) * 100}
						<div class="absolute top-0 flex flex-col items-center" style={`left: ${pct}%;`}>
							<div class="w-px h-2 bg-slate-600"></div>
							<span class="text-[9px] text-slate-500 font-mono mt-0.5 -translate-x-1/2">{sec}s</span>
						</div>
					{/each}
				</div>
				<span class="text-[9px] text-slate-500 font-mono flex-shrink-0 ml-2">{totalSec.toFixed(1)}s</span>
			{/if}
		</div>

		<!-- Track Row -->
		<div class="flex items-center gap-2">
			<!-- Filmstrip Track -->
			<!-- svelte-ignore a11y_click_events_have_key_events -->
			<div
				bind:this={trackContainer}
				class={`relative flex-1 h-14 rounded-md overflow-hidden cursor-pointer border-2 transition-colors select-none ${isTrimMode ? 'border-blue-500' : 'border-slate-600 hover:border-slate-500'}`}
				onclick={handleTrackClick}
				onmousemove={handleTrackMouseMove}
				onmouseleave={handleTrackMouseLeave}
				role="slider"
				aria-valuenow={currentFrame}
				aria-valuemin={0}
				aria-valuemax={totalFrames - 1}
				tabindex="0"
			>
				<!-- Thumbnail filmstrip background -->
				<div class="absolute inset-0 flex">
					{#if thumbnails.length > 0}
						{#each getThumbnailsForSegments() as thumb}
							<img
								src={thumb.src}
								alt=""
								class="h-full object-cover flex-shrink-0"
								style={`width: ${thumb.widthPercent}%;`}
								draggable="false"
							/>
						{/each}
					{:else}
						<div class="w-full h-full bg-slate-800 flex items-center justify-center">
							<span class="text-slate-500 text-xs">Loading thumbnails...</span>
						</div>
					{/if}
				</div>

				<!-- Darken overlay for contrast -->
				<div class="absolute inset-0 bg-black/20 pointer-events-none"></div>

				<!-- Trim range highlight -->
				{#if isTrimMode && trimStartFrame !== null && trimEndFrame !== null}
					<div
						class="absolute top-0 h-full bg-yellow-500/30 border-l-2 border-r-2 border-yellow-500 pointer-events-none"
						style={`left: ${(Math.min(trimStartFrame, trimEndFrame) / (totalFrames - 1)) * 100}%; right: ${100 - (Math.max(trimStartFrame, trimEndFrame) / (totalFrames - 1)) * 100}%; z-index: 3;`}
					></div>
				{/if}

				<!-- Trim start handle -->
				{#if isTrimMode && trimStartFrame !== null}
					<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
					<!-- svelte-ignore a11y_no_noninteractive_tabindex -->
					<div
						class="absolute top-0 h-full w-1.5 bg-yellow-400 cursor-col-resize hover:bg-yellow-300 transition-colors"
						style={`left: ${(trimStartFrame / (totalFrames - 1)) * 100}%; transform: translateX(-50%); z-index: 5;`}
						onmousedown={() => handleMouseDown('start')}
						role="separator"
						aria-label="Trim start"
						tabindex="0"
					>
						<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-3 h-6 bg-yellow-400 rounded-sm border border-yellow-300 shadow-lg flex items-center justify-center">
							<div class="w-0.5 h-3 bg-yellow-700 rounded-full"></div>
						</div>
					</div>
				{/if}

				<!-- Trim end handle -->
				{#if isTrimMode && trimEndFrame !== null}
					<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
					<!-- svelte-ignore a11y_no_noninteractive_tabindex -->
					<div
						class="absolute top-0 h-full w-1.5 bg-yellow-400 cursor-col-resize hover:bg-yellow-300 transition-colors"
						style={`left: ${(trimEndFrame / (totalFrames - 1)) * 100}%; transform: translateX(-50%); z-index: 5;`}
						onmousedown={() => handleMouseDown('end')}
						role="separator"
						aria-label="Trim end"
						tabindex="0"
					>
						<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-3 h-6 bg-yellow-400 rounded-sm border border-yellow-300 shadow-lg flex items-center justify-center">
							<div class="w-0.5 h-3 bg-yellow-700 rounded-full"></div>
						</div>
					</div>
				{/if}

				<!-- Hover indicator -->
				{#if trackHoverFrame !== null && !draggingTrimPoint}
					<div
						class="absolute top-0 h-full w-px bg-white/40 pointer-events-none"
						style={`left: ${(trackHoverFrame / (totalFrames - 1)) * 100}%; z-index: 4;`}
					></div>
					<!-- Hover timecode tooltip -->
					<div
						class="absolute -top-6 pointer-events-none"
						style={`left: ${(trackHoverFrame / (totalFrames - 1)) * 100}%; transform: translateX(-50%); z-index: 6;`}
					>
						<span class="bg-slate-800 border border-slate-600 text-[9px] font-mono text-slate-300 px-1.5 py-0.5 rounded whitespace-nowrap">
							{formatTimecode(trackHoverFrame)}
						</span>
					</div>
				{/if}

				<!-- Playhead -->
				{#if totalFrames > 0}
					<div
						class="absolute top-0 h-full pointer-events-none"
						style={`left: ${(currentFrame / (totalFrames - 1)) * 100}%; z-index: 6;`}
					>
						<!-- Playhead line -->
						<div class="absolute top-0 h-full w-0.5 bg-red-500 -translate-x-1/2"></div>
						<!-- Playhead top marker -->
						<div class="absolute -top-1.5 left-1/2 -translate-x-1/2 w-0 h-0"
							style="border-left: 5px solid transparent; border-right: 5px solid transparent; border-top: 6px solid #ef4444;"
						></div>
					</div>
				{/if}
			</div>
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
				<h2 class="text-lg font-bold text-white">
					{isTrimmingVideo ? 'Processing Trim...' : 'Confirm Trim'}
				</h2>
			</div>
			
			<!-- Content -->
			<div class="px-6 py-6 space-y-4">
				{#if isTrimmingVideo}
					<!-- Loading Spinner -->
					<div class="flex flex-col items-center justify-center py-8 space-y-4">
						<div class="w-12 h-12 border-4 border-slate-700 border-t-blue-500 rounded-full animate-spin"></div>
						<div class="text-center space-y-2">
							<p class="text-sm text-slate-300 font-medium">Trimming video segment...</p>
							<p class="text-xs text-slate-500">{trimProgress}% Complete</p>
						</div>
					</div>
				{:else}
					<!-- Trim Details -->
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
							<span class="text-slate-300 text-sm font-medium">Frames to Remove:</span>
							<span class="text-red-400 font-mono text-sm font-bold">{pendingTrimData.end - pendingTrimData.start + 1}</span>
						</div>
						<div class="flex justify-between items-center">
							<span class="text-slate-300 text-sm font-medium">Duration:</span>
							<span class="text-green-400 font-mono text-sm font-bold">{pendingTrimData.duration}s</span>
						</div>
					</div>
					
					<p class="text-slate-400 text-xs">This will permanently remove the selected frames from the video.</p>
				{/if}
			</div>
			
			<!-- Footer -->
			{#if !isTrimmingVideo}
				<div class="border-t border-slate-700 px-6 py-4 flex items-center gap-3 justify-end">
					<button
						onclick={dismissTrimModal}
						class="px-4 py-2 bg-slate-800 border border-slate-600 rounded-md text-slate-300 text-sm hover:bg-slate-700 transition-colors"
					>
						Cancel
					</button>
					
					<button
						onclick={confirmTrim}
						class="px-4 py-2 bg-red-600 border border-red-500 rounded-md text-white text-sm font-medium hover:bg-red-500 transition-colors"
					>
						Delete Frames
					</button>
				</div>
			{/if}
		</div>
	</div>
{/if}
