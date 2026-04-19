<script lang="ts">
	import VideoPlayer from '$lib/components/VideoPlayer.svelte';
</script>

<svelte:head>
	<title>Video Editor</title>
</svelte:head>

<div class="p-6 flex flex-col items-center min-h-screen">
	<div class="mb-6 text-center">
		<h2 class="text-2xl font-bold text-white mb-1">Video Editor</h2>
		<p class="text-slate-400 text-sm">Frame-accurate video scrubbing with WebCodecs API</p>
	</div>

	<div class="w-full max-w-2xl">
		<VideoPlayer src="/samples/test_video_01.mp4" />
	</div>
</div>
