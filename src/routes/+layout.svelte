<script lang="ts">
	import '../app.css';
	import { fade, fly } from 'svelte/transition';
	import { Keyboard as KeyboardIcon, Settings, Trophy, Code } from 'lucide-svelte';

	let { children } = $props();
</script>

<svelte:head>
	<title>sebeolsik — keyboard practice</title>
	<meta name="description" content="Master the Se-yeonsik 3-set keyboard layout with our premium typing practice application." />
	<link rel="icon" type="image/png" href="/favicon.png" />
	<link rel="preconnect" href="https://fonts.googleapis.com">
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous">
	<link href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;700&family=JetBrains+Mono:wght@400;700&family=Noto+Sans+KR:wght@300;400;500;700&display=swap" rel="stylesheet">
</svelte:head>

<div class="min-h-screen bg-[#111111] text-[#999999] selection:bg-[#333333] font-['Outfit']">
	<!-- Main Content -->
	<main class="w-full h-screen flex flex-col pt-8">
		{@render children()}
	</main>
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		overflow-y: scroll;
	}

	:global(::-webkit-scrollbar) {
		width: 8px;
	}

	:global(::-webkit-scrollbar-track) {
		background: #0a0a0c;
	}

	:global(::-webkit-scrollbar-thumb) {
		background: #27272a;
		border-radius: 10px;
	}

	:global(::-webkit-scrollbar-thumb:hover) {
		background: #3f3f46;
	}
</style>
