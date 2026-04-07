<script lang="ts">
	import '../app.css';
	import { fade, fly } from 'svelte/transition';
	import { Keyboard as KeyboardIcon, Settings, Trophy, Code, AirVent, Key } from 'lucide-svelte';

	let { children } = $props();
</script>

<svelte:head>
	<title>세벌식 세벌식 타자연습</title>
	<meta name="description" content="세벌식의 매력을 더 깊게, 세련된 인터페이스로 즐기는 타자 연습 서비스" />
	<meta property="og:title" content="세벌식 타자연습" />
	<meta property="og:description" content="세벌식의 매력을 더 깊게, 세련된 인터페이스로 즐기는 타자 연습 서비스" />
	<meta property="og:type" content="website" />
	<meta name="twitter:card" content="summary_large_image" />
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
