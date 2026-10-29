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
		qrX: 72.8,
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

	const STORAGE_KEY = 'ticm';

	let settings = $state<Settings>({ ...DEFAULTS });
	let settingsLoaded = $state(false);

	function loadSettings() {
		try {
			const raw = localStorage.getItem(STORAGE_KEY);
			if (!raw) return;
			const saved = JSON.parse(raw);
			if (!saved || typeof saved !== 'object') return;
			for (const key of Object.keys(DEFAULTS) as (keyof Settings)[]) {
				if (typeof saved[key] === typeof DEFAULTS[key]) {
					(settings as Record<string, unknown>)[key] = saved[key];
				}
			}
		} catch {}
	}

	let data = $state('');
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

	const safeName = (s: string) => s.replace(/[\\/:*?"<>|]/g, '').trim();
	const CLASS_RE = /^(XII|XI|X)\s+\S/i;
	const normalizeClass = (c: string) => c.replace(/\s+/g, ' ').trim().toUpperCase();

	type Row = { line: number; qr: string; name: string; cls: string };

	function splitColumns(line: string) {
		let cols = line
			.split('\t')
			.map((c) => c.trim())
			.filter(Boolean);
		if (cols.length < 3) {
			cols = line
				.split(/\s{2,}/)
				.map((c) => c.trim())
				.filter(Boolean);
		}
		return cols;
	}

	const parsed = $derived.by(() => {
		const rows: Row[] = [];
		const badFormat: number[] = [];
		const badClass: number[] = [];

		data.split(/\r?\n/).forEach((raw, i) => {
			if (!raw.trim()) return;
			const cols = splitColumns(raw);
			if (cols.length !== 3) {
				badFormat.push(i + 1);
				return;
			}
			const [qr, name, cls] = cols;
			if (!CLASS_RE.test(cls)) {
				badClass.push(i + 1);
				return;
			}
			rows.push({ line: i + 1, qr, name, cls: normalizeClass(cls) });
		});

		return { rows, badFormat, badClass };
	});

	const entries = $derived<Entry[]>(
		parsed.rows.map((r) => ({ name: `${r.name} ${r.cls}`, qr: r.qr }))
	);

	const classes = $derived([...new Set(parsed.rows.map((r) => r.cls))]);
	const className = $derived(classes.length === 1 ? classes[0] : '');
	const fmtLines = (l: number[]) =>
		l
			.slice(0, 5)
			.map((n) => `baris ${n}`)
			.join(', ') + (l.length > 5 ? ', ...' : '');

	const inputError = $derived.by(() => {
		if (!data.trim()) return '';

		if (parsed.badFormat.length)
			return `Format salah di ${fmtLines(parsed.badFormat)}. Tiap baris harus: KODE QR [Tab] NAMA [Tab] KELAS.`;

		if (parsed.badClass.length)
			return `Kelas tidak valid di ${fmtLines(parsed.badClass)}. Gunakan kelas X, XI, atau XII, mis. "X DPIB 1".`;

		if (classes.length > 1) {
			const odd = parsed.rows.filter((r) => r.cls !== classes[0]).map((r) => r.line);
			return `Kelas tidak boleh campur dalam satu bulk. Terdeteksi: ${classes.join(' / ')}. Yang berbeda dari "${classes[0]}": ${fmtLines(odd)}.`;
		}

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
		loadSettings();
		settingsLoaded = true;

		try {
			templateImg = await loadImage(DEFAULT_TEMPLATE);
		} catch {
			templateName = 'Belum ada template, silakan upload (PNG/WEBP).';
		}
	});

	async function onTemplateChange(e: Event) {
		const input = e.currentTarget as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		if (!ALLOWED_TYPES.includes(file.type)) {
			alert('Format template harus PNG atau WEBP.');
			input.value = '';
			return;
		}

		const url = URL.createObjectURL(file);
		try {
			templateImg = await loadImage(url);
			templateName = file.name;
		} catch (err) {
			alert('Something went wrong...');
			input.value = '';
		} finally {
			URL.revokeObjectURL(url);
		}
	}

	$effect(() => {
		const snapshot = $state.snapshot(settings) as Settings;
		if (!settingsLoaded) return;

		try {
			if (JSON.stringify(snapshot) === JSON.stringify(DEFAULTS)) {
				localStorage.removeItem(STORAGE_KEY);
			} else {
				localStorage.setItem(STORAGE_KEY, JSON.stringify(snapshot));
			}
		} catch {}
	});

	function resetSettings() {
		try {
			localStorage.clear();
		} catch {}
		Object.assign(settings, DEFAULTS);
		alert('Pengaturan direset.');
	}

	function reset(type: string) {
		const i = confirm(
			`Apa Anda yakin ingin melakukan ini?\n${type === 'reload' ? 'reload hanya akan me-refresh halaman, pengaturan tidak terpengaruh' : 'reset pengaturan akan menghapus pengaturan Teks Nama dan Kode QR, halaman tidak akan reload'}`
		);
		if (i) {
			if (type === 'reload') return window.location.reload();
			if (type === 'settings') return resetSettings();
		}
	}

	function resetInputs() {
		data = '';
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
		if (!data.trim()) return (error = 'Data masih kosong.');
		if (inputError) return (error = inputError);
		if (!entries.length) return (error = 'Tidak ada tiket yang valid.');

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
	<section class="hero">
		<h1>Ticket Creator Machine</h1>
		<p>Mesin pembuat tiket otomatis</p>
	</section>

	<section class="container">
		<form onsubmit={handleSubmit}>
			<fieldset>
				<legend>Petunjuk Penggunaan</legend>
				<div class="desc">
					<p><b>Buka website ini di komputer/laptop agar lebih mudah</b></p>
					<p>1. Upload desain atau template tiket dibagian Template Tiket</p>
					<p>2. Isi kolom Data dari data tiket spreadsheet</p>
					<p>3. Sesuaikan penempatan nama dan QR kode pada bagian Teks Nama dan QR Code</p>
					<p>4. Jika sudah sesuai semua, tekan pada Cetak Tiket</p>
					<p>5. Ulangi proses ini untuk semua kelas</p>
				</div>
			</fieldset>

			<div class="sepa"></div>

			<fieldset id="temp">
				<legend>Template Tiket</legend>
				<input type="file" accept="image/png,image/webp" onchange={onTemplateChange} />
				<small>Dipakai: {templateName}</small>
			</fieldset>

			<div class="inputs">
				<div>
					<label for="data">Data tiket (1 baris = 1 tiket)</label>
					<textarea
						name="data"
						id="data"
						rows="15"
						placeholder={'PBL-3HCN2YB1K4VIZ71NO\tADITYA SASANA MUKTI\tX TKP 1\nPBL-RLV5F4ZLGIHGXPDI1\tAKBAR FADHILLAH\tX TKP 1'}
						bind:value={data}></textarea>
					<button type="button" onclick={resetInputs} disabled={!data}> Reset Data </button>
				</div>
			</div>

			{#if data.trim()}
				<div class="summary">
					{#if inputError}
						<p class="error">{inputError}</p>
					{:else}
						<strong>{entries.length} tiket, kelas {className}</strong>
					{/if}
				</div>
			{/if}

			<fieldset style="border-color: red;">
				<legend style="color: red;">Perhatian!</legend>
				<div class="desc">
					<p>
						Format tiap baris (3 kolom dipisah Tab, bisa langsung paste dari spreadsheet):
						<b>KODE QR [Tab] NAMA [Tab] KELAS</b>
					</p>
					<p>Kelas yang didukung: X, XI, dan XII (mis. X DPIB 1, XI TJKT 2, XII TKP 1).</p>
					<p style="color: red;">Per bulk kelas harus sama, tidak boleh campur, misal:</p>
					<p><b>Boleh:</b></p>
					<p>PBL-3HCN2YB1K4VIZ71NO&nbsp;&nbsp;AYU ANDRA&nbsp;&nbsp;XI TJKT 2</p>
					<p>PBL-RLV5F4ZLGIHGXPDI1&nbsp;&nbsp;ALEX MORGAN&nbsp;&nbsp;XI TJKT 2</p>
					<p><b>Tidak boleh:</b></p>
					<p>PBL-3HCN2YB1K4VIZ71NO&nbsp;&nbsp;AYU ANDRA&nbsp;&nbsp;X DPIB 1</p>
					<p>PBL-IM2HILFAKY6BSF285&nbsp;&nbsp;ANGGA YUNANDA&nbsp;&nbsp;XII TJKT 2</p>
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
				<div class="btn-list">
					<button
						type="button"
						onclick={() => {
							reset('reload');
						}}>Reload</button
					>
					<button
						type="button"
						onclick={() => {
							reset('settings');
						}}>Reset Pengaturan</button
					>
				</div>
				<button type="submit" disabled={loading || !!inputError || !templateImg || !entries.length}>
					{loading ? 'Memproses...' : 'Cetak Tiket'}
				</button>
				{#if progress}<p class="progress">{progress}</p>{/if}
				{#if error && error !== inputError}<p class="error">{error}</p>{/if}
			</div>
		</form>
	</section>
</main>

<footer>
	<p>Dibuat oleh alumni. Ingin berkontribusi? silahkan ke</p>
	<a href="https://github.com/TheAlexSandro/TicketCreatorMachine" target="_blank"
		>TicketCreatorMachine</a
	>
	<p>GitHub repository.</p>
</footer>
