<script lang="ts">
	import { micromark } from 'micromark';
	import { gfmTable, gfmTableHtml } from 'micromark-extension-gfm-table';

	let html = $state('');

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

		return parsed;
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

	async function load() {
		const res = await fetch('https://garden.from.pub/dinner.md');
		const text = res.ok ? await res.text() : 'Failed to load.';
		html = parse(text);
	}

	load();
</script>

<!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
<main
	class="mx-auto prose max-w-4xl bg-white p-6"
	style="font-family: 'adriane', serif;"
	onclick={handleClick}
>
	{@html html}
</main>

<style>
	/* :global(body) {
		background-color: #eee;
	} */
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
		color: #1e3a5f;
		text-decoration: underline;
	}
	:global(.inline-expand) {
		overflow-y: auto;
	}
</style>
