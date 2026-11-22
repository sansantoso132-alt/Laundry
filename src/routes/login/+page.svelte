<script>
	import { supabase } from '$lib/supabaseClients.js';
	import { goto } from '$app/navigation';

	let mode = 'login';
	let name = '';
	let email = '';
	let password = '';
	let confirmPassword = '';
	let showPassword = false;
	let message = '';
	let messageType = '';
	let loading = false;

	// Kekuatan password (hanya tampilan, validasi tetap di handleSubmit)
	$: strength = getStrength(password);
	$: strengthLabel = ['', 'Lemah', 'Cukup', 'Kuat', 'Sangat kuat'][strength];

	function getStrength(p) {
		if (!p) return 0;
		let s = 0;
		if (p.length >= 8) s++;
		if (/[A-Z]/.test(p) && /[a-z]/.test(p)) s++;
		if (/\d/.test(p)) s++;
		if (/[^A-Za-z0-9]/.test(p) && p.length >= 8) s++;
		return Math.max(1, s);
	}

	function switchMode(newMode) {
		mode = newMode;
		message = '';
		messageType = '';
		password = '';
		confirmPassword = '';
	}

	function showMessage(text, type = 'error') {
		message = text;
		messageType = type;
	}

	// Baca role dari public.profiles, bukan dari input/form pengguna.
	async function redirectByRole(userId) {
		const { data: profile, error } = await supabase
			.from('profiles')
			.select('role')
			.eq('id', userId)
			.single();

		if (error) {
			throw new Error(`Tidak bisa membaca role akun: ${error.message}`);
		}

		if (profile.role === 'admin') {
			await goto('/admin');
			return;
		}

		if (profile.role === 'customer') {
			await goto('/dashboard');
			return;
		}

		throw new Error('Role akun tidak valid. Hubungi administrator.');
	}

	async function handleSubmit(event) {
		event.preventDefault();
		if (loading) return;

		message = '';
		messageType = '';
		const normalizedEmail = email.trim().toLowerCase();

		if (!normalizedEmail) return showMessage('Email wajib diisi.');
		if (mode === 'register') {
			if (!name.trim()) return showMessage('Nama lengkap wajib diisi.');
			if (password.length < 8) return showMessage('Password minimal 8 karakter.');
			if (password !== confirmPassword) return showMessage('Konfirmasi password tidak cocok.');
		} else if (!password) {
			return showMessage('Password wajib diisi.');
		}

		loading = true;
		try {
			if (mode === 'login') {
				const { data, error } = await supabase.auth.signInWithPassword({
					email: normalizedEmail,
					password
				});
				if (error) throw error;
				if (!data.user) throw new Error('Data pengguna tidak ditemukan.');
				showMessage('Login berhasil. Memeriksa akun...', 'success');
				await redirectByRole(data.user.id);
			} else {
				const { data, error } = await supabase.auth.signUp({
					email: normalizedEmail,
					password,
					options: {
						data: { full_name: name.trim() },
						emailRedirectTo: `${window.location.origin}/auth/callback`
					}
				});
				if (error) throw error;

				if (data.session && data.user) {
					showMessage('Pendaftaran berhasil. Mengalihkan...', 'success');
					await redirectByRole(data.user.id);
				} else {
					showMessage('Akun berhasil dibuat. Silakan verifikasi email, lalu masuk kembali.', 'success');
					password = '';
					confirmPassword = '';
				}
			}
		} catch (error) {
			console.error('Auth error:', error);
			showMessage(error?.message || 'Terjadi kesalahan. Silakan coba lagi.');
		} finally {
			loading = false;
		}
	}

	async function resetPassword() {
		if (!email.trim()) return showMessage('Masukkan email terlebih dahulu untuk reset password.');
		loading = true;
		message = '';
		try {
			const { error } = await supabase.auth.resetPasswordForEmail(email.trim().toLowerCase(), {
				redirectTo: `${window.location.origin}/reset-password`
			});
			if (error) throw error;
			showMessage('Link reset password dikirim. Periksa email kamu.', 'success');
		} catch (error) {
			showMessage(error?.message || 'Gagal mengirim link reset password.');
		} finally {
			loading = false;
		}
	}
</script>

<svelte:head>
	<title>{mode === 'login' ? 'Masuk' : 'Daftar'} | LaundryKu</title>
	<meta name="description" content="Masuk atau daftar akun LaundryKu" />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page">
	<div class="auth">
		<!-- ============ PANEL KIRI (brand) ============ -->
		<aside class="brand" aria-hidden="false">
			<div class="brand-bg"></div>

			<a class="logo" href="/" aria-label="LaundryKu">
				<span class="logo-mark">
					<svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3c3 4 6 6.5 6 10a6 6 0 0 1-12 0c0-3.5 3-6 6-10Z"/></svg>
				</span>
				<span>LaundryKu</span>
			</a>

			<div class="brand-main">
				<div class="machine" role="img" aria-label="Ilustrasi mesin cuci">
					<div class="m-body">
						<div class="m-panel">
							<span class="dot d1"></span><span class="dot d2"></span>
							<span class="knob"></span>
						</div>
						<div class="m-door">
							<div class="m-glass">
								<div class="water"><i></i><i></i></div>
								<span class="bub b1"></span><span class="bub b2"></span><span class="bub b3"></span><span class="bub b4"></span>
							</div>
						</div>
					</div>

					<div class="chip chip-a">
						<span class="chip-ico">
							<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6 9 17l-5-5"/></svg>
						</span>
						<div><strong>Cucian selesai</strong><small>Siap diambil hari ini</small></div>
					</div>
					<div class="chip chip-b">
						<span class="chip-ico alt">
							<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 2"/></svg>
						</span>
						<div><strong>Estimasi 24 jam</strong><small>Layanan ekspres</small></div>
					</div>
				</div>

				<h1>Laundry rapi,<br />tanpa antre.</h1>
				<p>Pesan, pantau, dan ambil cucian dari satu akun. Semua status pesanan ada di genggamanmu.</p>

				<ul class="points">
					<li><span class="tick">✓</span>Pesan kapan saja, di mana saja</li>
					<li><span class="tick">✓</span>Harga jelas sebelum diproses</li>
					<li><span class="tick">✓</span>Status cucian terpantau langsung</li>
				</ul>
			</div>

			<div class="brand-foot">© 2026 LaundryKu</div>
		</aside>

		<!-- ============ PANEL KANAN (form) ============ -->
		<main class="panel">
			<div class="wrap">
				<a class="m-logo" href="/" aria-label="LaundryKu">
					<span class="logo-mark">
						<svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3c3 4 6 6.5 6 10a6 6 0 0 1-12 0c0-3.5 3-6 6-10Z"/></svg>
					</span>
					<span>LaundryKu</span>
				</a>

				<header class="heading">
					<h2>{mode === 'login' ? 'Selamat datang kembali' : 'Buat akun baru'}</h2>
					<p>
						{mode === 'login'
							? 'Masuk untuk melihat dan mengelola pesanan laundry kamu.'
							: 'Daftar gratis dan mulai pesan laundry dalam hitungan menit.'}
					</p>
				</header>

				<div class="tabs" class:right={mode === 'register'} role="tablist" aria-label="Pilih Masuk atau Daftar">
					<span class="tab-thumb"></span>
					<button type="button" role="tab" aria-selected={mode === 'login'} class:active={mode === 'login'} onclick={() => switchMode('login')}>Masuk</button>
					<button type="button" role="tab" aria-selected={mode === 'register'} class:active={mode === 'register'} onclick={() => switchMode('register')}>Daftar</button>
				</div>

				<div aria-live="polite">
					{#if message}
						<div class="message" class:success={messageType === 'success'} class:error={messageType === 'error'} role={messageType === 'error' ? 'alert' : 'status'}>
							<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
								{#if messageType === 'success'}
									<circle cx="12" cy="12" r="9"/><path d="m8 12 3 3 5-6"/>
								{:else}
									<circle cx="12" cy="12" r="9"/><path d="M12 8v5M12 16h.01"/>
								{/if}
							</svg>
							<span>{message}</span>
						</div>
					{/if}
				</div>

				<form onsubmit={handleSubmit} novalidate>
					{#if mode === 'register'}
						<div class="field">
							<label for="name">Nama lengkap</label>
							<div class="control">
								<svg class="ico" viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="4"/><path d="M4 21c1-4 4-6 8-6s7 2 8 6"/></svg>
								<input id="name" type="text" autocomplete="name" bind:value={name} placeholder="Nama sesuai identitas" required />
							</div>
						</div>
					{/if}

					<div class="field">
						<label for="email">Alamat email</label>
						<div class="control">
							<svg class="ico" viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="5" width="18" height="14" rx="3"/><path d="m4 7 8 6 8-6"/></svg>
							<input id="email" type="email" autocomplete="email" bind:value={email} placeholder="nama@email.com" required />
						</div>
					</div>

					<div class="field">
						<div class="label-row">
							<label for="password">Password</label>
							{#if mode === 'login'}
								<button type="button" class="link small" onclick={resetPassword}>Lupa password?</button>
							{/if}
						</div>
						<div class="control">
							<svg class="ico" viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="10" width="16" height="11" rx="3"/><path d="M8 10V7a4 4 0 0 1 8 0v3"/></svg>
							<input
								id="password"
								type={showPassword ? 'text' : 'password'}
								autocomplete={mode === 'login' ? 'current-password' : 'new-password'}
								bind:value={password}
								placeholder={mode === 'login' ? 'Masukkan password' : 'Minimal 8 karakter'}
								required
							/>
							<button type="button" class="eye" aria-label={showPassword ? 'Sembunyikan password' : 'Tampilkan password'} aria-pressed={showPassword} onclick={() => (showPassword = !showPassword)}>
								{#if showPassword}
									<svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M3 3l18 18"/><path d="M10.6 6.1A10 10 0 0 1 12 6c5 0 8.5 4 9.5 6a14 14 0 0 1-3 3.7M6.5 6.8C4.3 8.2 2.8 10.3 2.5 12c1 2 4.5 6 9.5 6 1.5 0 2.8-.3 4-.9"/><path d="M9.9 9.9a3 3 0 0 0 4.2 4.2"/></svg>
								{:else}
									<svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M2.5 12C3.5 10 7 6 12 6s8.5 4 9.5 6c-1 2-4.5 6-9.5 6s-8.5-4-9.5-6Z"/><circle cx="12" cy="12" r="3"/></svg>
								{/if}
							</button>
						</div>

						{#if mode === 'register' && password}
							<div class="meter" aria-live="polite">
								<div class="bars s{strength}">
									<i></i><i></i><i></i><i></i>
								</div>
								<span>{strengthLabel}</span>
							</div>
						{/if}
					</div>

					{#if mode === 'register'}
						<div class="field">
							<label for="confirm">Konfirmasi password</label>
							<div class="control">
								<svg class="ico" viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3 5 6v5c0 4.5 3 8 7 10 4-2 7-5.5 7-10V6l-7-3Z"/><path d="m9 12 2 2 4-4"/></svg>
								<input id="confirm" type={showPassword ? 'text' : 'password'} autocomplete="new-password" bind:value={confirmPassword} placeholder="Ulangi password" required />
							</div>
						</div>

						<label class="terms">
							<input type="checkbox" required />
							<span class="box"><svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6 9 17l-5-5"/></svg></span>
							<span>Saya menyetujui syarat dan ketentuan yang berlaku.</span>
						</label>
					{/if}

					<button type="submit" class="submit" disabled={loading}>
						{#if loading}
							<span class="spinner" aria-hidden="true"></span>
							<span>Memproses...</span>
						{:else}
							<span>{mode === 'login' ? 'Masuk' : 'Buat akun'}</span>
							<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M5 12h14M13 6l6 6-6 6"/></svg>
						{/if}
					</button>
				</form>

				<div class="divider"><span>atau</span></div>

				<p class="switch">
					{mode === 'login' ? 'Belum punya akun?' : 'Sudah punya akun?'}
					<button type="button" class="link" onclick={() => switchMode(mode === 'login' ? 'register' : 'login')}>
						{mode === 'login' ? 'Daftar gratis' : 'Masuk'}
					</button>
				</p>

				<p class="secure">
					<svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="10" width="16" height="11" rx="3"/><path d="M8 10V7a4 4 0 0 1 8 0v3"/></svg>
					Data kamu dilindungi dengan enkripsi
				</p>
			</div>
		</main>
	</div>
</div>

<style>
	:global(*) { box-sizing: border-box; }
	:global(body) {
		margin: 0;
		font-family: 'Plus Jakarta Sans', Inter, 'Segoe UI', Arial, sans-serif;
		background: #f3f8fb;
		color: #0c2b45;
		-webkit-font-smoothing: antialiased;
	}
	:global(button), :global(input) { font: inherit; }

	.page {
		--ink: #0c2b45;
		--muted: #5b7186;
		--line: #dbe6ee;
		--teal: #0e7c86;
		--teal-dark: #0a6670;
		--aqua: #2bc4c0;
		--foam: #e7f6f6;
		--danger: #b42318;
		--ok: #0b7a4b;
		min-height: 100vh;
		min-height: 100dvh;
	}

	a { color: inherit; text-decoration: none; }

	:global(.page button:focus-visible),
	:global(.page input:focus-visible),
	:global(.page a:focus-visible) {
		outline: 3px solid rgba(14, 124, 134, 0.4);
		outline-offset: 2px;
	}

	.auth {
		min-height: 100vh;
		min-height: 100dvh;
		display: grid;
		grid-template-columns: minmax(0, 1.05fr) minmax(0, 1fr);
	}

	/* ================= BRAND ================= */
	.brand {
		position: relative;
		overflow: hidden;
		color: #fff;
		padding: 40px clamp(32px, 7vw, 96px);
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		background: #082238;
	}
	.brand-bg {
		position: absolute;
		inset: 0;
		background:
			radial-gradient(600px 420px at 85% 10%, rgba(43, 196, 192, 0.32), transparent 65%),
			radial-gradient(520px 520px at 0% 100%, rgba(14, 124, 134, 0.55), transparent 65%),
			linear-gradient(160deg, #0c2b45 0%, #082238 100%);
	}
	.brand::after {
		content: '';
		position: absolute;
		inset: 0;
		background-image: radial-gradient(rgba(255, 255, 255, 0.07) 1px, transparent 1px);
		background-size: 22px 22px;
		mask-image: linear-gradient(180deg, transparent, #000 40%, transparent);
		pointer-events: none;
	}
	.brand > * { position: relative; z-index: 1; }

	.logo { display: inline-flex; align-items: center; gap: 12px; font-weight: 800; font-size: 20px; letter-spacing: -0.3px; }
	.logo-mark {
		width: 40px; height: 40px; border-radius: 12px;
		display: grid; place-items: center;
		background: linear-gradient(145deg, var(--aqua), var(--teal));
		color: #fff;
		box-shadow: 0 8px 20px rgba(43, 196, 192, 0.35);
	}

	.brand-main { width: 100%; max-width: 520px; margin: 24px auto; }

	/* --- mesin cuci --- */
	.machine {
		position: relative;
		height: 300px;
		margin-bottom: 28px;
		display: grid;
		place-items: center;
	}
	.m-body {
		width: 200px; height: 250px;
		border-radius: 26px;
		background: linear-gradient(160deg, #ffffff 0%, #d9ecf2 100%);
		box-shadow: 0 40px 60px rgba(0, 0, 0, 0.35), inset 0 -6px 0 rgba(12, 43, 69, 0.06);
		padding: 16px;
		display: flex; flex-direction: column; align-items: center; gap: 16px;
	}
	.m-panel {
		width: 100%; height: 30px; border-radius: 10px;
		background: #e3f0f5;
		display: flex; align-items: center; gap: 7px; padding: 0 12px;
	}
	.dot { width: 8px; height: 8px; border-radius: 50%; background: #9db8c6; }
	.dot.d1 { background: var(--aqua); }
	.knob { margin-left: auto; width: 20px; height: 20px; border-radius: 50%; background: #fff; border: 3px solid #b5ccd8; }
	.m-door {
		width: 132px; height: 132px; border-radius: 50%;
		background: linear-gradient(145deg, #c9dde7, #8fb0c1);
		padding: 10px;
		box-shadow: inset 0 4px 10px rgba(12, 43, 69, 0.2);
	}
	.m-glass {
		position: relative; width: 100%; height: 100%;
		border-radius: 50%; overflow: hidden;
		background: #0b4a63;
		box-shadow: inset 0 6px 16px rgba(0, 0, 0, 0.4);
	}
	.water { position: absolute; left: -50%; top: 38%; width: 200%; height: 200%; }
	.water i {
		position: absolute; left: 0; width: 100%; height: 100%;
		border-radius: 42%;
		background: rgba(43, 196, 192, 0.75);
		animation: spin 6s linear infinite;
	}
	.water i:nth-child(2) { background: rgba(255, 255, 255, 0.22); border-radius: 38%; animation-duration: 9s; animation-direction: reverse; }
	.bub { position: absolute; border-radius: 50%; background: rgba(255, 255, 255, 0.75); bottom: -10px; animation: rise 4.5s ease-in infinite; }
	.b1 { width: 12px; height: 12px; left: 22%; animation-delay: 0s; }
	.b2 { width: 8px; height: 8px; left: 52%; animation-delay: 1.4s; }
	.b3 { width: 14px; height: 14px; left: 70%; animation-delay: 2.6s; }
	.b4 { width: 6px; height: 6px; left: 38%; animation-delay: 3.4s; }

	@keyframes spin { to { transform: rotate(360deg); } }
	@keyframes rise {
		0% { transform: translateY(0) scale(0.6); opacity: 0; }
		20% { opacity: 1; }
		100% { transform: translateY(-120px) scale(1); opacity: 0; }
	}
	@keyframes bob { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }

	.chip {
		position: absolute;
		display: flex; align-items: center; gap: 11px;
		padding: 11px 15px 11px 11px;
		border-radius: 14px;
		background: rgba(255, 255, 255, 0.96);
		color: var(--ink);
		box-shadow: 0 18px 36px rgba(0, 0, 0, 0.28);
		animation: bob 5s ease-in-out infinite;
	}
	.chip strong { display: block; font-size: 13px; }
	.chip small { display: block; font-size: 11px; color: var(--muted); margin-top: 2px; }
	.chip-ico {
		width: 34px; height: 34px; border-radius: 10px;
		display: grid; place-items: center;
		background: var(--foam); color: var(--teal);
	}
	.chip-ico.alt { background: #e8eefc; color: #3656c7; }
	.chip-a { left: 0; top: 20px; }
	.chip-b { right: 0; bottom: 18px; animation-delay: 1.8s; }

	.brand h1 {
		margin: 0 0 14px;
		font-size: clamp(30px, 3.2vw, 44px);
		line-height: 1.15;
		letter-spacing: -1.2px;
		font-weight: 800;
	}
	.brand-main > p { margin: 0 0 26px; color: #b9d2de; font-size: 15px; line-height: 1.75; max-width: 420px; }

	.points { list-style: none; margin: 0; padding: 0; display: grid; gap: 13px; }
	.points li { display: flex; align-items: center; gap: 12px; font-size: 14px; color: #e3f1f6; }
	.tick {
		width: 22px; height: 22px; border-radius: 50%;
		display: grid; place-items: center;
		background: rgba(43, 196, 192, 0.22); color: var(--aqua);
		font-size: 12px; font-weight: 800;
	}
	.brand-foot { font-size: 12px; color: #86a8b8; }

	/* ================= FORM ================= */
	.panel {
		display: flex; align-items: center; justify-content: center;
		padding: 48px 32px;
		background: #fff;
	}
	.wrap { width: 100%; max-width: 420px; }
	.m-logo { display: none; }

	.heading { margin-bottom: 26px; }
	.heading h2 { margin: 0 0 10px; font-size: 30px; letter-spacing: -0.9px; font-weight: 800; line-height: 1.15; }
	.heading p { margin: 0; color: var(--muted); font-size: 14.5px; line-height: 1.65; }

	/* tabs */
	.tabs {
		position: relative;
		display: grid; grid-template-columns: 1fr 1fr;
		padding: 4px; margin-bottom: 24px;
		background: #eef4f8; border-radius: 14px;
	}
	.tab-thumb {
		position: absolute; top: 4px; bottom: 4px; left: 4px;
		width: calc(50% - 4px);
		background: #fff; border-radius: 11px;
		box-shadow: 0 2px 10px rgba(12, 43, 69, 0.1);
		transition: transform 0.28s cubic-bezier(0.4, 0, 0.2, 1);
	}
	.tabs.right .tab-thumb { transform: translateX(100%); }
	.tabs button {
		position: relative; z-index: 1;
		border: 0; background: transparent; cursor: pointer;
		padding: 12px; border-radius: 11px;
		font-weight: 700; font-size: 14px; color: var(--muted);
		transition: color 0.2s;
	}
	.tabs button.active { color: var(--teal-dark); }

	/* message */
	.message {
		display: flex; gap: 10px; align-items: flex-start;
		padding: 12px 14px; margin-bottom: 20px;
		border-radius: 12px; font-size: 13.5px; line-height: 1.55;
		animation: pop 0.25s ease;
	}
	.message svg { flex: none; margin-top: 1px; }
	.message.success { color: var(--ok); background: #eafaf2; border: 1px solid #b5ead0; }
	.message.error { color: var(--danger); background: #fef3f2; border: 1px solid #fcd0cc; }
	@keyframes pop { from { opacity: 0; transform: translateY(-4px); } to { opacity: 1; transform: none; } }

	/* fields */
	.field { margin-bottom: 18px; }
	.field label, .label-row label { display: block; font-size: 13px; font-weight: 700; color: #243f57; margin-bottom: 8px; }
	.label-row { display: flex; justify-content: space-between; align-items: baseline; }
	.label-row label { margin-bottom: 8px; }

	.control { position: relative; }
	.control .ico {
		position: absolute; left: 14px; top: 50%; transform: translateY(-50%);
		color: #8aa0b2; pointer-events: none; transition: color 0.2s;
	}
	.control input {
		width: 100%; height: 52px;
		padding: 0 14px 0 44px;
		border: 1.5px solid var(--line); border-radius: 13px;
		background: #fbfdfe; color: var(--ink);
		font-size: 15px; /* 15px+ mencegah zoom otomatis di iOS saat fokus */
		outline: none;
		transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
	}
	.control input::placeholder { color: #9fb1c0; }
	.control input:hover { border-color: #c3d4e0; }
	.control input:focus { border-color: var(--teal); background: #fff; box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
	.control:focus-within .ico { color: var(--teal); }
	.control input[type='password'], .control #password { padding-right: 50px; }

	.eye {
		position: absolute; right: 6px; top: 50%; transform: translateY(-50%);
		width: 40px; height: 40px; border: 0; border-radius: 10px;
		display: grid; place-items: center;
		background: transparent; color: #7b92a5; cursor: pointer;
		transition: background 0.2s, color 0.2s;
	}
	.eye:hover { background: #eef4f8; color: var(--ink); }

	.link {
		border: 0; background: transparent; padding: 0; cursor: pointer;
		color: var(--teal); font-weight: 700; font-size: 13.5px;
	}
	.link:hover { color: var(--teal-dark); text-decoration: underline; text-underline-offset: 3px; }
	.link.small { font-size: 12.5px; }

	/* strength */
	.meter { display: flex; align-items: center; gap: 12px; margin-top: 10px; font-size: 12px; font-weight: 600; color: var(--muted); }
	.bars { flex: 1; display: grid; grid-template-columns: repeat(4, 1fr); gap: 5px; }
	.bars i { height: 5px; border-radius: 99px; background: #e4edf3; transition: background 0.25s; }
	.bars.s1 i:nth-child(-n + 1) { background: #e5484d; }
	.bars.s2 i:nth-child(-n + 2) { background: #f5a524; }
	.bars.s3 i:nth-child(-n + 3) { background: #2bb673; }
	.bars.s4 i { background: #0e9f6e; }

	/* terms */
	.terms { position: relative; display: flex; gap: 11px; align-items: flex-start; margin: 4px 0 22px; font-size: 13px; color: var(--muted); line-height: 1.55; cursor: pointer; }
	.terms input { position: absolute; opacity: 0; width: 20px; height: 20px; margin: 0; cursor: pointer; }
	.box {
		flex: none; width: 20px; height: 20px; border-radius: 6px;
		border: 1.5px solid #b9cad7; background: #fff; color: transparent;
		display: grid; place-items: center; margin-top: 1px;
		transition: all 0.18s;
	}
	.terms input:checked + .box { background: var(--teal); border-color: var(--teal); color: #fff; }
	.terms input:focus-visible + .box { outline: 3px solid rgba(14, 124, 134, 0.4); outline-offset: 2px; }

	/* submit */
	.submit {
		width: 100%; height: 54px; margin-top: 6px;
		border: 0; border-radius: 14px; cursor: pointer;
		display: flex; align-items: center; justify-content: center; gap: 10px;
		font-weight: 800; font-size: 15px; color: #fff;
		background: linear-gradient(135deg, var(--teal) 0%, #0b5f78 100%);
		box-shadow: 0 10px 22px rgba(14, 124, 134, 0.28);
		transition: transform 0.18s, box-shadow 0.18s, filter 0.18s;
	}
	.submit:hover:not(:disabled) { transform: translateY(-1px); box-shadow: 0 14px 28px rgba(14, 124, 134, 0.34); filter: brightness(1.04); }
	.submit:active:not(:disabled) { transform: translateY(0); }
	.submit:disabled { opacity: 0.75; cursor: wait; }
	.spinner {
		width: 18px; height: 18px; border-radius: 50%;
		border: 2.5px solid rgba(255, 255, 255, 0.35); border-top-color: #fff;
		animation: spin 0.7s linear infinite;
	}

	.divider { display: flex; align-items: center; gap: 14px; margin: 24px 0; color: #9fb1c0; font-size: 12px; }
	.divider::before, .divider::after { content: ''; flex: 1; height: 1px; background: #e6eef4; }

	.switch { margin: 0; text-align: center; color: var(--muted); font-size: 14px; }
	.switch .link { margin-left: 4px; font-size: 14px; }

	.secure {
		margin: 30px 0 0; display: flex; justify-content: center; align-items: center; gap: 7px;
		color: #8aa0b2; font-size: 12px;
	}

	/* ================= RESPONSIVE ================= */
	@media (max-width: 1100px) {
		.brand { padding-left: 36px; padding-right: 36px; }
		.machine { transform: scale(0.9); }
	}

	@media (max-width: 900px) {
		.auth { grid-template-columns: 1fr; }
		.brand { display: none; }
		.panel {
			align-items: flex-start;
			min-height: 100vh; min-height: 100dvh;
			padding: 0;
			background:
				radial-gradient(420px 260px at 90% -40px, rgba(43, 196, 192, 0.35), transparent 70%),
				linear-gradient(160deg, #0c2b45, #082238);
		}
		.wrap {
			max-width: 520px; margin: 0 auto;
			padding: 28px 24px calc(32px + env(safe-area-inset-bottom));
			display: flex; flex-direction: column;
			background: transparent;
		}
		.m-logo {
			display: inline-flex; align-items: center; gap: 11px;
			font-weight: 800; font-size: 19px; color: #fff;
			margin-bottom: 26px;
		}
		.m-logo .logo-mark { width: 36px; height: 36px; border-radius: 11px; }
		.heading h2, .heading p { color: #fff; }
		.heading p { color: #b9d2de; }
		.heading { margin-bottom: 22px; }

		/* Kartu putih untuk form di mobile */
		.tabs { background: rgba(255, 255, 255, 0.12); }
		.tabs button { color: #cfe3ec; }
		.tabs button.active { color: var(--teal-dark); }
		.tab-thumb { background: #fff; }

		form, .divider, .switch, .secure { position: relative; }
		form {
			background: #fff; border-radius: 20px;
			padding: 22px 20px 20px;
			box-shadow: 0 24px 50px rgba(0, 0, 0, 0.3);
		}
		.divider { color: #86a8b8; }
		.divider::before, .divider::after { background: rgba(255, 255, 255, 0.15); }
		.switch { color: #b9d2de; }
		.switch .link { color: var(--aqua); }
		.secure { color: #86a8b8; margin-top: 22px; }
	}

	@media (max-width: 480px) {
		.wrap { padding: 22px 16px calc(28px + env(safe-area-inset-bottom)); }
		.heading h2 { font-size: 25px; }
		.heading p { font-size: 14px; }
		form { padding: 20px 16px 18px; border-radius: 18px; }
		.control input { height: 50px; }
		.submit { height: 52px; }
	}

	@media (max-height: 640px) and (min-width: 901px) {
		.machine { display: none; }
	}

	@media (prefers-reduced-motion: reduce) {
		* { animation: none !important; transition: none !important; }
	}
</style>
