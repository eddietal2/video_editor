<script lang="ts">
	import { page } from '$app/stores';
</script>

<aside class="fixed left-0 top-0 h-screen w-64 bg-slate-950 border-r border-slate-800 flex flex-col">
	<div class="px-6 py-8 border-b border-slate-800">
		<h1 class="text-xl font-bold text-white">Video Editor</h1>
		<p class="text-xs text-slate-500 mt-2">WebCodecs Playground</p>
	</div>

	<nav class="flex-1 px-3 py-6 space-y-2 overflow-y-auto">
		<a
			href="/"
			class="flex items-center px-4 py-2.5 rounded-lg transition-colors duration-200 text-sm font-medium {$page.url.pathname === '/'
				? 'bg-slate-800 text-blue-400 border-l-2 border-blue-500'
				: 'text-slate-400 hover:text-slate-200 hover:bg-slate-900'}"
		>
			▶ Video Editor
		</a>

		<a
			href="/about"
			class="flex items-center px-4 py-2.5 rounded-lg transition-colors duration-200 text-sm font-medium {$page.url.pathname === '/about'
				? 'bg-slate-800 text-blue-400 border-l-2 border-blue-500'
				: 'text-slate-400 hover:text-slate-200 hover:bg-slate-900'}"
		>
			ℹ About
		</a>
	</nav>

	<div class="px-6 py-4 border-t border-slate-800">
		<p class="text-xs text-slate-600">v0.0.1</p>
	</div>
</aside>
