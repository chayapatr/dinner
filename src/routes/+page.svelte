<script lang="ts">
	import { micromark } from 'micromark';
	import { gfmTable, gfmTableHtml } from 'micromark-extension-gfm-table';
	import { parse as parseYaml } from 'yaml';
	import { onMount } from 'svelte';
	import Sun from '@lucide/svelte/icons/sun';
	import Moon from '@lucide/svelte/icons/moon';
	import Circle from '@lucide/svelte/icons/circle';
	import CircleDot from '@lucide/svelte/icons/circle-dot';

	let html = $state('');
	let headings: { id: string; text: string; level: number }[] = $state([]);
	let dark = $state(false);
	let high = $state(true);

	type Theme = 'light-low' | 'light-high' | 'dark-low' | 'dark-high';
	let theme: Theme = $derived(
		dark ? (high ? 'dark-high' : 'dark-low') : high ? 'light-high' : 'light-low'
	);

	function parse(text: string) {
		let parsed = micromark(text, {
			allowDangerousHtml: true,
			extensions: [gfmTable()],
			htmlExtensions: [gfmTableHtml()]
		});

		// external links first (http/https)
		parsed = parsed.replace(
			/<a href="((?:http|https):\/\/[^"]*)"/g,
			'<a href="$1" target="_blank" class="external-link"'
		);

		// internal links (everything else)
		parsed = parsed.replace(/<a href="(?!\w+:\/\/)([^"]*)"/g, (_: string, s: string) => {
			const slug = decodeURIComponent(s).replace(/^\//, '').replace(/\.md$/, '');
			return `<a href="#" data-slug="${slug}" class="internal-link"`;
		});

		// add ids to headings
		parsed = parsed.replace(
			/<(h[1-6])>(.*?)<\/h[1-6]>/g,
			(_: string, tag: string, text: string) => {
				const id = text
					.replace(/<[^>]*>/g, '')
					.trim()
					.toLowerCase()
					.replace(/\s+/g, '-');
				return `<${tag} id="${id}">${text}</${tag}>`;
			}
		);

		return parsed;
	}

	function extractHeadings(html: string) {
		const regex = /<h([1-6]) id="([^"]*)">(.*?)<\/h[1-6]>/g;
		const result: { id: string; text: string; level: number }[] = [];
		let match;
		while ((match = regex.exec(html)) !== null) {
			result.push({
				level: parseInt(match[1]),
				id: match[2],
				text: match[3].replace(/<[^>]*>/g, '')
			});
		}
		return result;
	}

	async function fetchAndInsert(slug: string, link: HTMLElement) {
		// toggle: if already expanded, close it
		const existing = link.parentElement?.querySelector(`.inline-expand[data-for="${slug}"]`);
		if (existing) {
			existing.remove();
			return;
		}

		const res = await fetch(`https://garden.from.pub/${encodeURIComponent(slug)}.md`);
		const text = res.ok ? await res.text() : 'Failed to load.';
		const parsed = parse(text);

		const wrapper = document.createElement('div');
		wrapper.className = 'inline-expand my-2 border border-neutral-300 rounded overflow-hidden';
		wrapper.dataset.for = slug;

		const header = document.createElement('div');
		header.className =
			'flex items-center justify-between px-3 py-1 bg-neutral-100 border-b border-neutral-300 text-sm text-neutral-600';

		const title = document.createElement('span');
		title.textContent = slug + '.md';

		const close = document.createElement('button');
		close.textContent = 'x';
		close.className = 'cursor-pointer text-neutral-400 hover:text-neutral-700';
		close.onclick = () => wrapper.remove();

		header.appendChild(title);
		header.appendChild(close);
		wrapper.appendChild(header);

		const content = document.createElement('div');
		content.className = 'prose p-4';
		content.innerHTML = parsed;
		wrapper.appendChild(content);

		// insert after the parent <p> or after the link itself
		const parent = link.closest('p') || link;
		parent.after(wrapper);
	}

	function handleClick(e: MouseEvent) {
		const internal = (e.target as HTMLElement).closest('.internal-link') as HTMLElement | null;
		if (internal?.dataset.slug) {
			e.preventDefault();
			fetchAndInsert(internal.dataset.slug, internal);
		}
	}

	function extractFrontmatter(raw: string) {
		const match = raw.match(/^---\n([\s\S]*?)\n---\n?([\s\S]*)$/);
		if (!match) return { meta: {}, body: raw };
		return { meta: parseYaml(match[1]) as Record<string, unknown>, body: match[2] };
	}

	async function load() {
		const res = await fetch('https://garden.from.pub/dinner.md');
		const raw = res.ok ? await res.text() : 'Failed to load.';
		const { meta, body } = extractFrontmatter(raw);
		if (Array.isArray(meta.rolling)) rolling = meta.rolling;
		html = parse(body);
		headings = extractHeadings(html);
	}

	let rolling: string[] = $state([]);
	const marqueeText = $derived(
		rolling.length > 0 ? Array(10).fill(rolling.join(' ⋅ ')).join(' ⋅ ') : ''
	);

	onMount(() => load());
</script>

<!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
<div class="flex theme-{theme}" style="font-family: 'adriane', serif;" onclick={handleClick}>
	<div class="fixed top-2.5 right-2.5 z-10 flex flex-col gap-2 md:flex-row">
		<button
			class="cursor-pointer text-neutral-400 hover:text-neutral-900"
			onclick={() => (dark = !dark)}
		>
			{#if dark}
				<Sun size={20} />
			{:else}
				<Moon size={20} />
			{/if}
		</button>
		<button
			class="cursor-pointer text-neutral-400 hover:text-neutral-900"
			onclick={() => (high = !high)}
		>
			{#if high}
				<Circle size={20} />
			{:else}
				<CircleDot size={20} />
			{/if}
		</button>
	</div>
	<nav class="fixed top-0 left-0 hidden h-screen w-56 overflow-y-auto p-5 text-sm md:block">
		<ul class="space-y-1">
			{#each headings.filter((h) => h.level > 1) as h}
				<li style="padding-left: {(h.level - 2) * 0.75}rem">
					<a href="#{h.id}" class="text-neutral-500 no-underline hover:text-neutral-900">{h.text}</a
					>
				</li>
			{/each}
		</ul>
	</nav>
	<main class="mx-auto prose max-w-3xl min-w-0 flex-1 bg-white p-6 pb-12">
		{@html html}
	</main>
</div>

{#if rolling.length > 0}
	<div
		class="marquee-bar fixed right-0 bottom-0 left-0 z-10 overflow-hidden py-1 text-sm"
		class:marquee-dark={!dark}
		class:marquee-light={dark}
	>
		<div class="marquee-content whitespace-nowrap">
			{marqueeText}
		</div>
	</div>
{/if}

<style>
	:global(h1),
	:global(h2),
	:global(h3) {
		font-family: 'discordia', sans-serif;
	}
	:global(.internal-link) {
		text-decoration: none;
		color: inherit;
	}
	:global(.internal-link)::before {
		content: '[[';
	}
	:global(.internal-link)::after {
		content: ']]';
	}
	:global(.external-link) {
		color: #4a7ab5;
		text-decoration: underline;
	}
	:global(.inline-expand) {
		overflow-y: auto;
	}
	/* table layout */
	:global(table) {
		table-layout: fixed;
		width: 100%;
		border: none !important;
		border-collapse: separate;
		border-spacing: 4px;
	}
	:global(thead),
	:global(tbody),
	:global(tr),
	:global(th),
	:global(td) {
		border: none !important;
		margin: 0 !important;
		padding: 0 !important;
	}
	:global(table td),
	:global(table th) {
		overflow: hidden;
		border-radius: 8px;
	}
	:global(table img) {
		width: 100%;
		display: block;
		margin: 0 !important;
		max-height: 320px;
		object-fit: cover;
	}
	/* light low */
	.theme-light-low {
		background-color: #f5f0e8;
		color: #5c4f3a;
	}
	.theme-light-low :global(.prose) {
		color: #5c4f3a;
	}
	.theme-light-low :global(h1) {
		color: #4a3f2e;
	}
	.theme-light-low :global(h2) {
		color: #4a3f2e;
	}
	.theme-light-low :global(h3) {
		color: #4a3f2e;
	}
	.theme-light-low :global(main) {
		background-color: #f5f0e8;
	}
	.theme-light-low :global(.external-link) {
		color: #7a9abf;
	}
	.theme-light-low :global(.inline-expand div:first-child) {
		background-color: #ece6da;
		border-color: #d5cdbf;
		color: #8a7d68;
	}
	.theme-light-low :global(.inline-expand) {
		border-color: #d5cdbf;
	}
	.theme-light-low :global(code) {
		background-color: #ece6da;
	}
	.theme-light-low :global(pre) {
		background-color: #ece6da !important;
	}

	/* light high (default — no overrides needed except main bg) */
	.theme-light-high :global(main) {
		background-color: white;
	}

	/* dark low */
	.theme-dark-low {
		background-color: #252830;
		color: #b0b8c4;
	}
	.theme-dark-low :global(.prose) {
		color: #b0b8c4;
	}
	.theme-dark-low :global(h1) {
		color: #c8cdd8;
	}
	.theme-dark-low :global(h2) {
		color: #c8cdd8;
	}
	.theme-dark-low :global(h3) {
		color: #c8cdd8;
	}
	.theme-dark-low :global(main) {
		background-color: #252830;
	}
	.theme-dark-low :global(.internal-link) {
		color: #b0b8c4;
	}
	.theme-dark-low :global(.external-link) {
		color: #8aaac8;
	}
	.theme-dark-low :global(nav a) {
		color: #6b7280;
	}
	.theme-dark-low :global(nav a:hover) {
		color: #b0b8c4;
	}
	.theme-dark-low :global(.inline-expand) {
		border-color: #3a3f4a;
	}
	.theme-dark-low :global(.inline-expand div:first-child) {
		background-color: #2e3240;
		border-color: #3a3f4a;
		color: #6b7280;
	}
	.theme-dark-low :global(code) {
		background-color: #2e3240;
		color: #b0b8c4;
	}
	.theme-dark-low :global(pre) {
		background-color: #2e3240 !important;
	}

	/* dark high */
	.theme-dark-high {
		background-color: #1a1a1a;
		color: #e0e0e0;
	}
	.theme-dark-high :global(.prose) {
		color: #e0e0e0;
	}
	.theme-dark-high :global(h1) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(h2) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(h3) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(h4) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(h5) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(h6) {
		color: #f0f0f0;
	}
	.theme-dark-high :global(main) {
		background-color: #1a1a1a;
	}
	.theme-dark-high :global(.internal-link) {
		color: #e0e0e0;
	}
	.theme-dark-high :global(.external-link) {
		color: #7ba4d4;
	}
	.theme-dark-high :global(nav a) {
		color: #999;
	}
	.theme-dark-high :global(nav a:hover) {
		color: #ccc;
	}
	.theme-dark-high :global(.inline-expand) {
		border-color: #444;
	}
	.theme-dark-high :global(.inline-expand div:first-child) {
		background-color: #2a2a2a;
		border-color: #444;
		color: #aaa;
	}
	.theme-dark-high :global(code) {
		background-color: #2a2a2a;
		color: #e0e0e0;
	}
	.theme-dark-high :global(pre) {
		background-color: #2a2a2a !important;
	}

	/* marquee */
	.marquee-dark {
		background-color: #1a1a1a;
		color: #e0e0e0;
	}
	.marquee-light {
		background-color: white;
		color: #333;
	}
	.marquee-content {
		/* font-family: sans-serif; */
		display: inline-block;
		animation: marquee 150s linear infinite;
	}
	@keyframes marquee {
		0% {
			transform: translateX(0);
		}
		100% {
			transform: translateX(-50%);
		}
	}
</style>
