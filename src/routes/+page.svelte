<script lang="ts">
	import { onMount } from 'svelte';
	import QRCode from 'qrcode';
	import JSZip from 'jszip';

	const DEFAULT_TEMPLATE = '/ticket-template.png';
	const ALLOWED_TYPES = ['image/png', 'image/webp'];

	const DEFAULTS = {
		fontFamily: 'Arial',
		bold: true,
		textColor: '#ffffff',
		strokeColor: '#000000',
		strokeWidth: 0,
		fontSize: 12,
		textY: 90,
		maxTextWidth: 20,
		qrSize: 61,
		qrX: 72.87,
		qrY: 11,
		qrMargin: 1,
		qrDark: '#000000',
		qrLight: '#ffffff',
		qrErrorLevel: 'M' as 'L' | 'M' | 'Q' | 'H',
		qrBorderColor: '#ffffff',
		qrBorderWidth: 0
	};
	type Settings = typeof DEFAULTS;
	type Entry = { name: string; qr: string };

	let settings = $state<Settings>({ ...DEFAULTS });

	let identifier = $state('');
	let qrContent = $state('');
	let loading = $state(false);
	let progress = $state('');
	let error = $state('');

	let templateImg = $state<HTMLImageElement | null>(null);
	let templateName = $state('Default (static/ticket-template.png)');
	let previewCanvas = $state<HTMLCanvasElement | null>(null);
	let renderId = 0;

	const FONTS = [
		'Arial',
		'Helvetica',
		'Verdana',
		'Tahoma',
		'Trebuchet MS',
		'Georgia',
		'Times New Roman',
		'Courier New',
		'Impact'
	];

	const toLines = (v: string) =>
		v
			.split(/\r?\n/)
			.map((l) => l.trim())
			.filter(Boolean);

	const safeName = (s: string) => s.replace(/[\\/:*?"<>|]/g, '').trim();
	const CLASS_RE = /\s(XII|XI|X)\s+(\S.*)$/i;
	const NO_CLASS = 'Tanpa Kelas';

	function detectClass(name: string) {
		const m = name.match(CLASS_RE);
		return m ? `${m[1].toUpperCase()} ${m[2].trim().toUpperCase()}` : NO_CLASS;
	}

	const names = $derived(toLines(identifier));
	const qrs = $derived(toLines(qrContent));
	const entries = $derived<Entry[]>(
		names.map((name, i) => ({
			name,
			qr: qrs.length ? (qrs[i] ?? '') : name
		}))
	);
	const classes = $derived([...new Set(names.map(detectClass))]);
	const className = $derived(classes.length === 1 && classes[0] !== NO_CLASS ? classes[0] : '');
	const classError = $derived.by(() => {
		if (!names.length) return '';

		if (classes.length > 1) {
			const base = classes[0];
			const odd = names
				.map((n, i) => ({ n, line: i + 1 }))
				.filter((x) => detectClass(x.n) !== base);
			const sample = odd
				.slice(0, 5)
				.map((x) => `baris ${x.line}`)
				.join(', ');
			return `Kelas tidak boleh campur dalam satu bulk. Terdeteksi: ${classes.join(' / ')}. Yang berbeda dari "${base}": ${sample}${odd.length > 5 ? ', ...' : ''}.`;
		}

		if (classes[0] === NO_CLASS)
			return 'Kelas tidak terdeteksi. Format tiap baris: NAMA X DPIB 1 (kelas X, XI, atau XII).';

		return '';
	});

	function loadImage(src: string) {
		return new Promise<HTMLImageElement>((resolve, reject) => {
			const img = new Image();
			img.onload = () => resolve(img);
			img.onerror = () => reject(new Error('Gagal memuat gambar template.'));
			img.src = src;
		});
	}

	onMount(async () => {
		try {
			templateImg = await loadImage(DEFAULT_TEMPLATE);
		} catch {
			templateName = 'Belum ada template, silakan upload';
		}
	});

	async function onTemplateChange(e: Event) {
		const input = e.currentTarget as HTMLInputElement;
		const file = input.files?.[0];
		error = '';
		if (!file) return;

		if (!ALLOWED_TYPES.includes(file.type)) {
			error = 'Format template harus PNG atau WEBP.';
			input.value = '';
			return;
		}

		const url = URL.createObjectURL(file);
		try {
			templateImg = await loadImage(url);
			templateName = file.name;
		} catch (err) {
			error = err instanceof Error ? err.message : 'Template tidak valid.';
			input.value = '';
		} finally {
			URL.revokeObjectURL(url);
		}
	}

	function resetSettings() {
		Object.assign(settings, DEFAULTS);
	}

	function resetInputs() {
		identifier = '';
		qrContent = '';
		error = '';
		progress = '';
	}

	async function makeQr(text: string, s: Settings, H: number) {
		const size = Math.max(10, Math.round((H * s.qrSize) / 100));
		const c = document.createElement('canvas');
		await QRCode.toCanvas(c, text, {
			width: size,
			margin: s.qrMargin,
			errorCorrectionLevel: s.qrErrorLevel,
			color: { dark: s.qrDark, light: s.qrLight }
		});
		return c;
	}

	function paint(
		canvas: HTMLCanvasElement,
		template: HTMLImageElement,
		name: string,
		qrCanvas: HTMLCanvasElement,
		s: Settings
	) {
		const W = template.naturalWidth;
		const H = template.naturalHeight;
		canvas.width = W;
		canvas.height = H;
		const ctx = canvas.getContext('2d')!;
		ctx.drawImage(template, 0, 0);

		// QR
		const qrSize = qrCanvas.width;
		const cx = (W * s.qrX) / 100;
		const qx = Math.round(cx - qrSize / 2);
		const qy = Math.round((H * s.qrY) / 100);
		ctx.drawImage(qrCanvas, qx, qy);

		// Border QR
		const bw = (H * s.qrBorderWidth) / 100;
		if (bw > 0) {
			ctx.strokeStyle = s.qrBorderColor;
			ctx.lineWidth = bw;
			ctx.strokeRect(qx + bw / 2, qy + bw / 2, qrSize - bw, qrSize - bw);
		}

		// Teks nama
		const weight = s.bold ? 'bold' : 'normal';
		let fontPx = (H * s.fontSize) / 100;
		ctx.textAlign = 'center';
		ctx.textBaseline = 'alphabetic';
		ctx.lineJoin = 'round';
		ctx.font = `${weight} ${fontPx}px "${s.fontFamily}", sans-serif`;

		const maxW = (W * s.maxTextWidth) / 100;
		const textW = ctx.measureText(name).width;
		if (textW > maxW) {
			fontPx *= maxW / textW;
			ctx.font = `${weight} ${fontPx}px "${s.fontFamily}", sans-serif`;
		}

		const ty = (H * s.textY) / 100;
		if (s.strokeWidth > 0) {
			ctx.strokeStyle = s.strokeColor;
			ctx.lineWidth = (fontPx * s.strokeWidth) / 100;
			ctx.strokeText(name, cx, ty);
		}
		ctx.fillStyle = s.textColor;
		ctx.fillText(name, cx, ty);
	}

	$effect(() => {
		const img = templateImg;
		const canvas = previewCanvas;
		const s = $state.snapshot(settings) as Settings;
		const first = entries[0];
		const name = first?.name ?? 'NAMA SISWA X DPIB 1';
		const qr = first?.qr || 'PREVIEW';
		if (!img || !canvas) return;

		const id = ++renderId;
		makeQr(qr, s, img.naturalHeight)
			.then((qrCanvas) => {
				if (id !== renderId) return;
				paint(canvas, img, name, qrCanvas, s);
			})
			.catch(() => {});
	});

	function download(blob: Blob, filename: string) {
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = filename;
		a.click();
		setTimeout(() => URL.revokeObjectURL(url), 1000);
	}

	async function fillZip(
		target: JSZip,
		list: Entry[],
		template: HTMLImageElement,
		s: Settings,
		onTick: () => void
	) {
		const used = new Set<string>();

		for (let i = 0; i < list.length; i++) {
			const { name, qr } = list[i];
			const qrCanvas = await makeQr(qr, s, template.naturalHeight);
			const canvas = document.createElement('canvas');
			paint(canvas, template, name, qrCanvas, s);

			const blob = await new Promise<Blob>((resolve, reject) =>
				canvas.toBlob(
					(b) => (b ? resolve(b) : reject(new Error('Gagal membuat gambar'))),
					'image/png'
				)
			);

			let file = safeName(name) || `tiket-${i + 1}`;
			if (used.has(file)) file = `${file} (${i + 1})`;
			used.add(file);

			target.file(`${file}.png`, blob);
			onTick();
		}
	}

	async function handleSubmit(e: SubmitEvent) {
		e.preventDefault();
		error = '';

		if (!templateImg) return (error = 'Template belum ada. Upload dulu (PNG/WEBP).');
		if (!names.length) return (error = 'Identifier masih kosong.');
		if (classError) return (error = classError);
		if (qrs.length && qrs.length !== names.length)
			return (error = `Jumlah baris tidak sama: Identifier ${names.length}, Isi QR ${qrs.length}. Kosongkan Isi QR kalau mau pakai identifier sebagai isi QR.`);

		loading = true;
		try {
			const s = $state.snapshot(settings) as Settings;
			const total = entries.length;
			let done = 0;

			const zip = new JSZip();
			await fillZip(zip, entries, templateImg, s, () => {
				done++;
				progress = `Memproses ${done}/${total}...`;
			});

			const blob = await zip.generateAsync({ type: 'blob' });
			download(blob, `${safeName(className)}.zip`);
			progress = `Selesai: ${total} tiket (${className}).`;
		} catch (err) {
			error = err instanceof Error ? err.message : 'Terjadi kesalahan.';
			progress = '';
		} finally {
			loading = false;
		}
	}
</script>

{#snippet num(label: string, key: keyof Settings, min: number, max: number, step: number)}
	<label class="field">
		<span>{label}</span>
		<input type="number" {min} {max} {step} bind:value={settings[key] as number} />
	</label>
{/snippet}

{#snippet color(label: string, key: keyof Settings)}
	<label class="field">
		<span>{label}</span>
		<input type="color" bind:value={settings[key] as string} />
	</label>
{/snippet}

<main>
	<h1>Ticket Creator Machine</h1>

	<section class="container">
		<form onsubmit={handleSubmit}>
			<fieldset>
				<legend>Template</legend>
				<input type="file" accept="image/png,image/webp" onchange={onTemplateChange} />
				<small>Dipakai: {templateName}</small>
			</fieldset>

			<div class="inputs">
				<div>
					<label for="identifier">Identifier (1 baris = 1 tiket, kelas harus sama)</label>
					<textarea
						name="identifier"
						id="identifier"
						cols="50"
						rows="15"
						placeholder={'AYU ANDRA X DPIB 1\nANADA X DPIB 1'}
						bind:value={identifier}></textarea>
					<button type="button" onclick={resetInputs} disabled={!identifier && !qrContent}>
						Reset Identifier &amp; Isi QR
					</button>
				</div>

				<div>
					<label for="qr"
						>Isi QR (urutan sama dengan Identifier, kosongkan = pakai identifier)</label
					>
					<textarea name="qr" id="qr" cols="50" rows="15" bind:value={qrContent}></textarea>
				</div>
			</div>

			{#if names.length}
				<div class="summary">
					{#if classError}
						<p class="error">{classError}</p>
					{:else}
						<strong>{names.length} tiket, kelas {className}</strong>
					{/if}
				</div>
			{/if}

			<fieldset style="border-color: red;">
				<legend style="color: red;">Perhatian!</legend>
				<div class="desc">
					<p style="color: red;">Per bulk kelas harus sama, tidak boleh campur, misal:</p>
					<b>Boleh:</b>
					<p>AYU ANDRA X DPIB 1</p>
					<p>ALEX MORGAN X DPIB 1</p>
					<p>BAYU SADEWA X DPIB 1</p>
					<b>Tidak boleh:</b>
					<p>AYU ANDRA X DPIB 1</p>
					<p>ALEX MORGAN X DPIB 1</p>
					<p>ANGGA YUNANDA X TJKT 2</p>
				</div>
			</fieldset>

			<fieldset>
				<legend>Teks Nama</legend>
				<div class="grid">
					<label class="field">
						<span>Font</span>
						<select bind:value={settings.fontFamily}>
							{#each FONTS as f}<option value={f}>{f}</option>{/each}
						</select>
					</label>
					<label class="field check">
						<input type="checkbox" bind:checked={settings.bold} />
						<span>Bold</span>
					</label>
					{@render color('Warna teks', 'textColor')}
					{@render color('Warna stroke', 'strokeColor')}
					{@render num('Tebal stroke (% font)', 'strokeWidth', 0, 50, 1)}
					{@render num('Ukuran font (%)', 'fontSize', 1, 50, 0.5)}
					{@render num('Posisi Y teks (%)', 'textY', 0, 100, 0.5)}
					{@render num('Lebar maks teks (%)', 'maxTextWidth', 1, 100, 0.5)}
				</div>
			</fieldset>

			<fieldset>
				<legend>QR Code</legend>
				<div class="grid">
					{@render num('Ukuran (%)', 'qrSize', 5, 100, 0.5)}
					{@render num('Posisi X tengah (%)', 'qrX', 0, 100, 0.1)}
					{@render num('Posisi Y atas (%)', 'qrY', 0, 100, 0.5)}
					{@render num('Margin (modul)', 'qrMargin', 0, 10, 1)}
					{@render color('Warna QR', 'qrDark')}
					{@render color('Warna latar QR', 'qrLight')}
					{@render color('Warna border QR', 'qrBorderColor')}
					{@render num('Tebal border QR (%)', 'qrBorderWidth', 0, 20, 0.1)}
					<label class="field">
						<span>Error correction</span>
						<select bind:value={settings.qrErrorLevel}>
							<option value="L">L (rendah)</option>
							<option value="M">M</option>
							<option value="Q">Q</option>
							<option value="H">H (tinggi)</option>
						</select>
					</label>
				</div>
			</fieldset>

			<div class="preview">
				<span>Preview</span>
				{#if templateImg}
					<canvas bind:this={previewCanvas}></canvas>
				{:else}
					<p>Upload template (PNG/WEBP) untuk melihat preview.</p>
				{/if}
			</div>

			<div class="submit">
				<button type="button" onclick={resetSettings}>Reset Pengaturan</button>
				<button type="button" onclick={resetInputs} disabled={!identifier && !qrContent}>
					Reset Identifier &amp; Isi QR
				</button>
				<button type="submit" disabled={loading || !!classError}>
					{loading ? 'Memproses...' : 'Cetak Tiket'}
				</button>
				{#if progress}<p class="progress">{progress}</p>{/if}
				{#if error && error !== classError}<p class="error">{error}</p>{/if}
			</div>
		</form>
	</section>
</main>
