<script lang="ts">
	import * as pdfjsLib from 'pdfjs-dist';
	import workerUrl from 'pdfjs-dist/build/pdf.worker.mjs?url';

	pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl;

	type Tab = 'native' | 'pdfjs';

	let activeTab: Tab = $state('native');
	let fileUrl: string | null = $state(null);
	let fileName = $state('');
	let canvasContainer: HTMLDivElement | undefined = $state();
	let pdfStatus = $state('');

	function handleFile(e: Event) {
		const file = (e.target as HTMLInputElement).files?.[0];
		if (!file) return;
		if (fileUrl) URL.revokeObjectURL(fileUrl);
		fileUrl = URL.createObjectURL(file);
		fileName = file.name;
	}

	function useSample() {
		if (fileUrl && !fileUrl.startsWith('/')) URL.revokeObjectURL(fileUrl);
		fileUrl = '/sample.pdf';
		fileName = 'sample.pdf';
	}

	async function renderWithPdfJs(url: string, container: HTMLDivElement) {
		pdfStatus = 'Loading...';
		container.innerHTML = '';
		try {
			const pdf = await pdfjsLib.getDocument({ url }).promise;
			for (let i = 1; i <= pdf.numPages; i++) {
				const page = await pdf.getPage(i);

				// Fit to container on narrow screens, but never upscale beyond natural size on desktop
				const containerWidth = container.clientWidth || window.innerWidth;
				const baseViewport = page.getViewport({ scale: 1 });
				const scale = Math.min(containerWidth / baseViewport.width, 1);
				const viewport = page.getViewport({ scale });

				const canvas = document.createElement('canvas');
				canvas.width = viewport.width;
				canvas.height = viewport.height;
				canvas.style.display = 'block';
				canvas.style.width = '100%';
				canvas.style.marginBottom = '12px';
				canvas.style.borderRadius = '4px';
				canvas.style.boxShadow = '0 1px 4px rgba(0,0,0,0.12)';
				container.appendChild(canvas);
				await page.render({ canvas, viewport }).promise;
			}
			pdfStatus = `${pdf.numPages} page(s) rendered`;
		} catch (err) {
			pdfStatus = `Error: ${err instanceof Error ? err.message : String(err)}`;
		}
	}

	$effect(() => {
		if (activeTab === 'pdfjs' && fileUrl && canvasContainer) {
			renderWithPdfJs(fileUrl, canvasContainer);
		}
	});
</script>

<main class="min-h-screen bg-gray-50">
	<!-- Header -->
	<div class="px-4 pb-4 pt-6 sm:px-6">
		<h1 class="mb-0.5 text-xl font-semibold text-gray-800 sm:text-2xl">PDF Renderer Test</h1>
		<p class="text-sm text-gray-500">Upload a PDF and compare native browser rendering vs PDF.js</p>
	</div>

	<!-- File upload -->
	<div class="px-4 pb-4 sm:px-6">
		<label
			class="flex cursor-pointer items-center gap-3 rounded-xl border-2 border-dashed border-gray-300 bg-white p-4 transition active:border-blue-400 sm:gap-4 sm:p-5"
		>
			<div class="min-w-0 flex-1">
				{#if fileName}
					<p class="truncate text-sm font-medium text-gray-800">{fileName}</p>
					<p class="text-xs text-gray-400">Tap to change</p>
				{:else}
					<p class="text-sm font-medium text-gray-600">Tap to upload a PDF</p>
					<p class="text-xs text-gray-400">Only .pdf files are accepted</p>
				{/if}
			</div>
			<span
				class="shrink-0 rounded-lg bg-blue-600 px-4 py-2.5 text-sm font-medium text-white active:bg-blue-700"
			>
				{fileName ? 'Change' : 'Choose PDF'}
			</span>
			<input type="file" accept="application/pdf" class="hidden" onchange={handleFile} />
		</label>
		<div class="mt-2 flex items-center gap-2">
			<div class="h-px flex-1 bg-gray-200"></div>
			<span class="text-xs text-gray-400">or</span>
			<div class="h-px flex-1 bg-gray-200"></div>
		</div>
		<button
			class="mt-2 w-full rounded-xl border border-gray-200 bg-white py-3 text-sm font-medium text-gray-700 transition active:bg-gray-50"
			onclick={useSample}
		>
			Use sample PDF
		</button>
	</div>

	{#if fileUrl}
		<!-- Sticky tabs -->
		<div class="sticky top-0 z-10 border-b border-gray-200 bg-white shadow-sm">
			<div class="flex">
				<button
					class="flex flex-1 items-center justify-center py-4 text-sm font-medium transition {activeTab ===
					'native'
						? 'border-b-2 border-blue-600 text-blue-600'
						: 'text-gray-500'}"
					onclick={() => (activeTab = 'native')}
				>
					Native (iframe)
				</button>
				<button
					class="flex flex-1 items-center justify-center py-4 text-sm font-medium transition {activeTab ===
					'pdfjs'
						? 'border-b-2 border-blue-600 text-blue-600'
						: 'text-gray-500'}"
					onclick={() => (activeTab = 'pdfjs')}
				>
					PDF.js
				</button>
			</div>
		</div>

		<!-- Tab content -->
		{#if activeTab === 'native'}
			<div class="px-4 py-4 sm:px-6">
				<p class="mb-3 text-xs text-gray-400">
					Uses the browser's built-in PDF viewer via &lt;iframe&gt;
				</p>
				<iframe
					src={fileUrl}
					class="h-[80vh] w-full rounded-lg border border-gray-200"
					title="Native PDF viewer"
				></iframe>
			</div>
		{:else}
			<div class="px-4 py-4 sm:px-6">
				<p class="mb-4 text-xs text-gray-400">
					Rendered page-by-page on &lt;canvas&gt; using PDF.js
					{#if pdfStatus}
						— <span
							class="font-medium {pdfStatus.startsWith('Error') ? 'text-red-500' : 'text-gray-600'}"
							>{pdfStatus}</span
						>
					{/if}
				</p>
				<div bind:this={canvasContainer} class="w-full overflow-hidden"></div>
			</div>
		{/if}
	{:else}
		<div class="px-4 sm:px-6">
			<div class="rounded-xl bg-white p-16 text-center text-sm text-gray-400 shadow-sm">
				Upload a PDF above to begin testing
			</div>
		</div>
	{/if}
</main>
