<script>
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabaseClients.js';
	import { goto } from '$app/navigation';

	let status = 'Memverifikasi email...';
	let failed = false;
	onMount(async () => {
		try {
			const url = new URL(window.location.href);
			const code = url.searchParams.get('code');
			const tokenHash = url.searchParams.get('token_hash');
			const type = url.searchParams.get('type');
			if (code) {
				const { error } = await supabase.auth.exchangeCodeForSession(code);
				if (error) throw error;
			} else if (tokenHash && type) {
				const { error } = await supabase.auth.verifyOtp({ token_hash: tokenHash, type });
				if (error) throw error;
			}
			const { data, error } = await supabase.auth.getSession();
			if (error) throw error;
			if (data.session && type === 'recovery') await goto('/reset-password');
			else if (data.session) {
				status = 'Email terverifikasi. Mengalihkan ke dashboard...';
				await goto('dashboard');
			} else status = 'Email berhasil diverifikasi. Silakan login.';
		} catch (e) { failed = true; status = e?.message || 'Verifikasi gagal. Coba minta link baru.'; }
	});
</script>
<svelte:head><title>Verifikasi | LaundryKu</title></svelte:head>
<main style="min-height:100vh;display:grid;place-items:center;font-family:Arial,sans-serif;background:#f4f7fb;padding:24px">
	<div style="background:white;padding:32px;border-radius:16px;max-width:480px;text-align:center;box-shadow:0 10px 40px #0001">
		<h1>LaundryKu</h1><p>{status}</p><a href="/">Kembali ke login</a>
	</div>
</main>
