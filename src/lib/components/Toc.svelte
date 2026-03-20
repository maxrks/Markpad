<script lang="ts">
	let { markdownBody, htmlContent, onBeforeJump } = $props<{
		markdownBody: HTMLElement | null;
		htmlContent: string;
		onBeforeJump?: () => void;
	}>();

	interface TocItem {
		id: string;
		text: string;
		level: number;
		isBlock: boolean;
	}

	let items = $state<TocItem[]>([]);
	let activeId = $state<string | null>(null);
	let tocContainer: HTMLElement | null = $state(null);
	let activeTargetEl: HTMLElement | null = null;
	
	// when user clicks a toc entry, lock active id until scroll catches up
	let clickLock: string | null = null;
	let clickLockTimer: ReturnType<typeof setTimeout> | null = null;

	$effect(() => {
		if (htmlContent && markdownBody) {
			const result: TocItem[] = [];

			const hs = markdownBody.querySelectorAll('h1, h2, h3, h4, h5, h6') as NodeListOf<HTMLElement>;
			for (const h of Array.from(hs)) {
				let text = h.textContent || '';
				text = text.replace(/\s*\^[a-zA-Z0-9_-]+$/, '');
				const anchor = h.querySelector('a.anchor') as HTMLElement | null;
				const id = h.id || (anchor ? anchor.id : '');
				if (id) {
					result.push({ id, text: text.trim(), level: parseInt(h.tagName[1], 10), isBlock: false });
				}
			}

			const blockAnchors = markdownBody.querySelectorAll('a[id].block-id-anchor, span[id].block-id-anchor') as NodeListOf<HTMLElement>;
			for (const el of Array.from(blockAnchors)) {
				const id = el.id;
				const label = el.getAttribute('data-label') || id;
				result.push({ id, text: label, level: 0, isBlock: true });
			}

			const allIds = new Map<string, number>();
			const allEls = markdownBody.querySelectorAll('[id]') as NodeListOf<HTMLElement>;
			let order = 0;
			for (const el of Array.from(allEls)) {
				allIds.set(el.id, order++);
			}
			result.sort((a, b) => (allIds.get(a.id) ?? 999) - (allIds.get(b.id) ?? 999));

			items = result;
		} else {
			items = [];
		}
	});

	function handleScroll() {
		if (!markdownBody || items.length === 0) return;
		
		if (!clickLock && activeTargetEl) {
			activeTargetEl.classList.remove('toc-target-active');
			activeTargetEl = null;
		}

		if (clickLock) return;

		const containerRect = markdownBody.getBoundingClientRect();
		let currentActive = items[0]?.id || null;

		for (const item of items) {
			const el = markdownBody.querySelector(`[id="${CSS.escape(item.id)}"]`);
			if (el) {
				const rect = el.getBoundingClientRect();
				if (rect.top - containerRect.top < 150) {
					currentActive = item.id;
				} else {
					break;
				}
			}
		}

		if (activeId !== currentActive) {
			activeId = currentActive;
			scrollTocIntoView();
		}
	}

	function scrollTocIntoView() {
		if (tocContainer && activeId) {
			const activeEl = tocContainer.querySelector(`[data-id="${CSS.escape(activeId)}"]`);
			if (activeEl) activeEl.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
		}
	}

	$effect(() => {
		if (markdownBody) {
			markdownBody.addEventListener('scroll', handleScroll, { passive: true });
			return () => markdownBody.removeEventListener('scroll', handleScroll);
		}
	});

	function jumpTo(id: string) {
		const el = markdownBody?.querySelector(`[id="${CSS.escape(id)}"]`) as HTMLElement | null;
		if (el && markdownBody) {
			onBeforeJump?.();
			// lock active id immediately so scroll handler doesn't override
			clickLock = id;
			activeId = id;
			scrollTocIntoView();

			// highlight element persistently until scroll
			if (activeTargetEl) activeTargetEl.classList.remove('toc-target-active');
			el.classList.add('toc-target-active');
			activeTargetEl = el;

			const containerRect = markdownBody.getBoundingClientRect();
			const elRect = el.getBoundingClientRect();
			const targetScrollTop = elRect.top - containerRect.top + markdownBody.scrollTop - 60;
			markdownBody.scrollTo({ top: targetScrollTop, behavior: 'smooth' });

			// release lock after scroll settles
			if (clickLockTimer) clearTimeout(clickLockTimer);
			clickLockTimer = setTimeout(() => { clickLock = null; }, 600);
		}
	}
</script>

<div class="toc-container" bind:this={tocContainer}>
	{#if items.length > 0}
		<ul class="toc-list">
			{#each items as item}
				<li class="toc-item {item.isBlock ? 'block-item' : `level-${item.level}`}">
					{#if item.isBlock}
					<button
						class="toc-link toc-block {activeId === item.id ? 'active' : ''}"
						data-id={item.id}
						onclick={() => jumpTo(item.id)}>
						<svg class="block-icon" viewBox="0 0 24 24" width="12" height="12" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<path d="M19 21l-7-5-7 5V5a2 2 0 0 1 2-2h10a2 2 0 0 1 2 2z"/>
						</svg>
						{item.text}
					</button>
					{:else}
						<button
							class="toc-link {activeId === item.id ? 'active' : ''}"
							data-id={item.id}
							onclick={() => jumpTo(item.id)}>
							{item.text}
						</button>
					{/if}
				</li>
			{/each}
		</ul>
	{:else}
		<div class="toc-empty">No headings found</div>
	{/if}
</div>

<style>
	.toc-container {
		width: 240px;
		flex-shrink: 0;
		height: 100%;
		background-color: var(--color-canvas-default);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		font-family: var(--win-font);
	}

	.toc-list {
		margin: 0;
		padding: 16px 0;
		list-style: none;
		overflow-y: auto;
		flex: 1;
		direction: rtl; /* move scrollbar to left */
	}

	.toc-empty {
		padding: 16px;
		color: var(--color-fg-muted);
		font-size: 13px;
		text-align: center;
	}

	.toc-item {
		margin: 1px 0;
		direction: ltr; /* keep text content ltr */
	}

	.toc-link {
		display: block;
		width: 100%;
		text-align: left;
		background: none;
		border: none;
		padding: 3px 16px;
		color: var(--color-fg-muted);
		font-size: 13px;
		cursor: pointer;
		transition: color 0.1s ease, font-weight 0.1s ease;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		font-family: inherit;
		line-height: 1.5;
	}

	.toc-link:hover {
		color: var(--color-fg-default);
	}

	.toc-link.active {
		color: var(--color-fg-default);
		font-weight: 600;
	}

	.toc-block {
		display: flex;
		align-items: center;
		gap: 5px;
		padding-left: 16px;
		font-size: 12.5px;
	}

	.block-icon {
		flex-shrink: 0;
		opacity: 0.4;
	}

	.level-1 .toc-link { padding-left: 16px; font-weight: 500; font-size: 13px; }
	.level-2 .toc-link { padding-left: 28px; }
	.level-3 .toc-link { padding-left: 40px; font-size: 12.5px; }
	.level-4 .toc-link { padding-left: 52px; font-size: 12px; opacity: 0.9; }
	.level-5 .toc-link { padding-left: 64px; font-size: 12px; opacity: 0.8; }
	.level-6 .toc-link { padding-left: 76px; font-size: 12px; opacity: 0.7; }
</style>
