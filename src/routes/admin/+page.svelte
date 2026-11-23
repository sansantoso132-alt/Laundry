<script>
import { supabase } from '$lib/supabaseClient';
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { supabase } from '$lib/supabaseClient.js';

    let loading = true;
    let refreshing = false;
    let saving = false;

    let errorMessage = '';
    let warningMessage = '';
    let notice = '';
    let noticeTimer;

    let user = null;
    let activeTab = 'pesanan';
    let search = '';
    let statusFilter = 'Semua';
    let paymentFilter = 'Semua';

    let orders = [];
    let profiles = [];
    let messages = [];
    let vouchers = [];
    let feedbacks = [];

    let selectedOrder = null;
    let selectedMessage = null;
    let proofUrl = '';
    let proofLoading = false;
    let proofError = '';

    let voucherForm = null; // null = modal tertutup

    const orderStatuses = ['Menunggu', 'Diproses', 'Siap Diambil', 'Selesai', 'Dibatalkan'];
    const paymentStatuses = ['Belum Dibayar', 'Menunggu Verifikasi', 'Bayar di Tempat', 'Lunas', 'Ditolak'];
    const methodLabel = { qris: 'QRIS', transfer: 'Transfer bank', cod: 'Bayar di tempat' };
    const tabTitles = {
        pesanan: 'Pesanan customer',
        voucher: 'Voucher & promo',
        ulasan: 'Ulasan customer',
        pesan: 'Pesan customer'
    };

    // ---------- statistik ----------
    $: waitingCount = orders.filter((o) => o.status === 'Menunggu').length;
    $: processingCount = orders.filter((o) => o.status === 'Diproses').length;
    $: readyCount = orders.filter((o) => o.status === 'Siap Diambil').length;
    $: verifyCount = orders.filter((o) => o.payment_status === 'Menunggu Verifikasi' && o.status !== 'Dibatalkan').length;
    $: revenue = orders
        .filter((o) => o.payment_status === 'Lunas' && o.status !== 'Dibatalkan')
        .reduce((sum, o) => sum + Number(o.total_amount || 0), 0);
    $: unreadCount = messages.filter((m) => !m.is_read).length;

    $: avgRating = feedbacks.length ? feedbacks.reduce((s, f) => s + f.rating, 0) / feedbacks.length : 0;
    $: aspectAvg = {
        clean: avg(feedbacks.map((f) => f.cleanliness)),
        time: avg(feedbacks.map((f) => f.timeliness)),
        service: avg(feedbacks.map((f) => f.service))
    };
    $: activeVoucherCount = vouchers.filter((v) => v.is_active && !isExpired(v)).length;

    function avg(list) {
        const nums = list.filter((n) => typeof n === 'number');
        return nums.length ? nums.reduce((a, b) => a + b, 0) / nums.length : 0;
    }
    function isExpired(v) {
        return !!v.valid_until && v.valid_until < new Date().toISOString().slice(0, 10);
    }

    // ---------- lookup ----------
    function customerFor(order) {
        return profiles.find((p) => p.id === order.customer_id);
    }
    function orderFor(id) {
        return orders.find((o) => o.id === id);
    }

    // ---------- filter ----------
    $: filteredOrders = orders.filter((order) => {
        const q = search.toLowerCase();
        const customer = customerFor(order);

        const matchesSearch = [
            order.order_number,
            order.service_name,
            order.customer_id,
            order.voucher_code,
            customer?.full_name,
            customer?.email
        ].some((value) => String(value || '').toLowerCase().includes(q));

        const matchesStatus = statusFilter === 'Semua' || order.status === statusFilter;
        const matchesPayment = paymentFilter === 'Semua' || order.payment_status === paymentFilter;

        return matchesSearch && matchesStatus && matchesPayment;
    });

    $: filteredMessages = messages.filter((message) => {
        const q = search.toLowerCase();
        return [message.name, message.email, message.subject, message.message].some((value) =>
            String(value || '').toLowerCase().includes(q)
        );
    });

    $: filteredVouchers = vouchers.filter((v) => {
        const q = search.toLowerCase();
        return [v.code, v.title, v.description].some((value) => String(value || '').toLowerCase().includes(q));
    });

    $: filteredFeedbacks = feedbacks.filter((f) => {
        const q = search.toLowerCase();
        const order = orderFor(f.order_id);
        const customer = profiles.find((p) => p.id === f.customer_id);
        return [order?.order_number, customer?.full_name, customer?.email, f.comment].some((value) =>
            String(value || '').toLowerCase().includes(q)
        );
    });

    // ---------- format ----------
    function rupiah(value) {
        return new Intl.NumberFormat('id-ID', {
            style: 'currency',
            currency: 'IDR',
            maximumFractionDigits: 0
        }).format(Number(value || 0));
    }

    function dateFormat(value) {
        if (!value) return '-';
        return new Date(value).toLocaleString('id-ID', { dateStyle: 'medium', timeStyle: 'short' });
    }

    function dayFormat(value) {
        if (!value) return '-';
        return new Date(value + 'T12:00:00').toLocaleDateString('id-ID', {
            day: 'numeric',
            month: 'short',
            year: 'numeric'
        });
    }

    function shortDay(value) {
        if (!value) return '-';
        return new Date(value + 'T12:00:00').toLocaleDateString('id-ID', { day: 'numeric', month: 'short' });
    }

    function stars(n) {
        const v = Math.round(Number(n || 0));
        return '★'.repeat(v) + '☆'.repeat(5 - v);
    }

    function voucherValue(v) {
        return v.discount_type === 'percent' ? `${Number(v.discount_value)}%` : rupiah(v.discount_value);
    }

    function notify(text) {
        notice = text;
        errorMessage = '';
        clearTimeout(noticeTimer);
        noticeTimer = setTimeout(() => (notice = ''), 4000);
    }

    function statusClass(status) {
        return (
            {
                Menunggu: 'waiting',
                Diproses: 'processing',
                'Siap Diambil': 'ready',
                Selesai: 'completed',
                Dibatalkan: 'cancelled'
            }[status] || 'waiting'
        );
    }

    function payClass(status) {
        return (
            {
                Lunas: 'completed',
                'Menunggu Verifikasi': 'processing',
                'Bayar di Tempat': 'ready',
                'Belum Dibayar': 'waiting',
                Ditolak: 'cancelled',
                Gagal: 'cancelled'
            }[status] || 'waiting'
        );
    }

    function initials(email) {
        return (email || 'A').trim().charAt(0).toUpperCase();
    }

    function setTab(tab) {
        activeTab = tab;
        search = '';
    }

    function closeModals() {
        if (saving) return;
        selectedOrder = null;
        selectedMessage = null;
        voucherForm = null;
    }

    // =====================================
    // PEMERIKSAAN LOGIN ADMIN
    // =====================================

    async function init() {
        loading = true;
        errorMessage = '';

        try {
            const { data: authData, error: authError } = await supabase.auth.getUser();
            if (authError) throw authError;

            if (!authData.user) {
                await goto('/login');
                return;
            }

            user = authData.user;

            const { data: profile, error: profileError } = await supabase
                .from('profiles')
                .select('id, role')
                .eq('id', user.id)
                .maybeSingle();

            if (profileError) throw profileError;

            if (!profile || profile.role !== 'admin') {
                await goto('/dashboard');
                return;
            }

            await loadData();
        } catch (error) {
            console.error('Admin initialization:', error);
            errorMessage = error.message || 'Gagal membuka dashboard admin.';
        } finally {
            loading = false;
        }
    }

    // =====================================
    // MENGAMBIL DATA SUPABASE
    // =====================================

    async function loadData() {
        errorMessage = '';
        warningMessage = '';

        try {
            const { data: orderData, error: orderError } = await supabase
                .from('customer_orders')
                .select('*')
                .order('created_at', { ascending: false });

            if (orderError) throw orderError;
            orders = orderData || [];

            const customerIds = [...new Set(orders.map((order) => order.customer_id).filter(Boolean))];

            if (customerIds.length > 0) {
                const { data: profileData, error: profileError } = await supabase
                    .from('profiles')
                    .select('id, full_name, email, role')
                    .in('id', customerIds);

                if (profileError) throw profileError;
                profiles = profileData || [];
            } else {
                profiles = [];
            }

            const { data: messageData, error: messageError } = await supabase
                .from('customer_messages')
                .select('*')
                .order('created_at', { ascending: false });

            if (messageError) {
                console.error('Pesan customer:', messageError);
                messages = [];
            } else {
                messages = messageData || [];
            }

            // voucher & ulasan: jangan gagalkan halaman bila tabel belum dibuat
            const warnings = [];

            const { data: voucherData, error: voucherError } = await supabase
                .from('vouchers')
                .select('*')
                .order('created_at', { ascending: false });
            if (voucherError) {
                console.error('Voucher:', voucherError);
                vouchers = [];
                warnings.push('voucher');
            } else {
                vouchers = voucherData || [];
            }

            const { data: feedbackData, error: feedbackError } = await supabase
                .from('order_feedback')
                .select('*')
                .order('created_at', { ascending: false });
            if (feedbackError) {
                console.error('Ulasan:', feedbackError);
                feedbacks = [];
                warnings.push('ulasan');
            } else {
                feedbacks = feedbackData || [];
            }

            if (warnings.length) {
                warningMessage = `Data ${warnings.join(' & ')} belum bisa dimuat. Jalankan laundryku_fitur_baru.sql dan laundryku_admin.sql di Supabase.`;
            }
        } catch (error) {
            console.error('Load data:', error);
            errorMessage = error.message || 'Gagal mengambil data Supabase.';
        }
    }

    async function refreshData() {
        if (refreshing) return;
        refreshing = true;
        await loadData();
        refreshing = false;
        if (!errorMessage) notify('Data berhasil diperbarui.');
    }

    // =====================================
    // PESANAN & PEMBAYARAN
    // =====================================

    async function editOrder(order) {
        selectedOrder = { ...order };
        proofUrl = '';
        proofError = '';

        if (order.payment_proof_url) {
            proofLoading = true;
            const { data, error } = await supabase.storage
                .from('payment-proofs')
                .createSignedUrl(order.payment_proof_url, 600);
            proofLoading = false;

            if (error) {
                proofError = 'Bukti tidak bisa dimuat: ' + error.message;
            } else if (selectedOrder?.id === order.id) {
                proofUrl = data.signedUrl;
            }
        }
    }

    async function saveOrder() {
        if (!selectedOrder || saving) return;

        saving = true;
        errorMessage = '';

        try {
            const update = {
                status: selectedOrder.status,
                payment_status: selectedOrder.payment_status,
                updated_at: new Date().toISOString()
            };

            // catat waktu pelunasan
            if (selectedOrder.payment_status === 'Lunas') {
                update.paid_at = selectedOrder.paid_at || new Date().toISOString();
            } else {
                update.paid_at = null;
            }

            const { error } = await supabase.from('customer_orders').update(update).eq('id', selectedOrder.id);
            if (error) throw error;

            selectedOrder = null;
            await loadData();
            if (!errorMessage) notify('Pesanan berhasil diperbarui!');
        } catch (error) {
            console.error('Update order:', error);
            errorMessage = error.message || 'Gagal memperbarui pesanan.';
        } finally {
            saving = false;
        }
    }

    // =====================================
    // VOUCHER
    // =====================================

    function newVoucher() {
        voucherForm = {
            id: null,
            code: '',
            title: '',
            description: '',
            discount_type: 'percent',
            discount_value: '',
            min_order: 0,
            max_discount: '',
            valid_until: '',
            is_active: true
        };
    }

    function editVoucher(v) {
        voucherForm = {
            id: v.id,
            code: v.code,
            title: v.title,
            description: v.description || '',
            discount_type: v.discount_type,
            discount_value: v.discount_value,
            min_order: v.min_order ?? 0,
            max_discount: v.max_discount ?? '',
            valid_until: v.valid_until || '',
            is_active: v.is_active
        };
    }

    async function saveVoucher() {
        if (!voucherForm || saving) return;

        const code = voucherForm.code.trim().toUpperCase().replace(/\s+/g, '');
        const value = Number(voucherForm.discount_value);

        if (!code) return (errorMessage = 'Kode voucher wajib diisi.');
        if (!voucherForm.title.trim()) return (errorMessage = 'Judul promo wajib diisi.');
        if (!value || value <= 0) return (errorMessage = 'Nilai diskon harus lebih dari 0.');
        if (voucherForm.discount_type === 'percent' && value > 100) return (errorMessage = 'Diskon persen maksimal 100.');

        saving = true;
        errorMessage = '';

        const payload = {
            code,
            title: voucherForm.title.trim(),
            description: voucherForm.description.trim() || null,
            discount_type: voucherForm.discount_type,
            discount_value: value,
            min_order: Number(voucherForm.min_order || 0),
            max_discount: voucherForm.max_discount === '' ? null : Number(voucherForm.max_discount),
            valid_until: voucherForm.valid_until || null,
            is_active: voucherForm.is_active
        };

        const query = voucherForm.id
            ? supabase.from('vouchers').update(payload).eq('id', voucherForm.id)
            : supabase.from('vouchers').insert(payload);

        const { error } = await query;
        saving = false;

        if (error) {
            errorMessage = error.code === '23505' ? 'Kode voucher sudah dipakai.' : error.message;
            return;
        }

        voucherForm = null;
        await loadData();
        if (!errorMessage) notify('Voucher berhasil disimpan.');
    }

    async function toggleVoucher(v) {
        const { error } = await supabase.from('vouchers').update({ is_active: !v.is_active }).eq('id', v.id);
        if (error) {
            errorMessage = error.message;
            return;
        }
        vouchers = vouchers.map((x) => (x.id === v.id ? { ...x, is_active: !v.is_active } : x));
        notify(v.is_active ? `Voucher ${v.code} dinonaktifkan.` : `Voucher ${v.code} diaktifkan.`);
    }

    async function deleteVoucher(v) {
        if (!confirm(`Hapus voucher ${v.code}?`)) return;

        const { error } = await supabase.from('vouchers').delete().eq('id', v.id);
        if (error) {
            errorMessage = error.message;
            return;
        }
        voucherForm = null;
        await loadData();
        if (!errorMessage) notify('Voucher dihapus.');
    }

    // =====================================
    // PESAN CUSTOMER
    // =====================================

    async function openMessage(message) {
        selectedMessage = message;

        if (!message.is_read) {
            const { error } = await supabase.from('customer_messages').update({ is_read: true }).eq('id', message.id);

            if (error) {
                errorMessage = error.message;
                return;
            }

            messages = messages.map((m) => (m.id === message.id ? { ...m, is_read: true } : m));
            selectedMessage = { ...message, is_read: true };
        }
    }

    async function deleteMessage() {
        if (!selectedMessage) return;
        if (!confirm('Yakin ingin menghapus pesan ini?')) return;

        const { error } = await supabase.from('customer_messages').delete().eq('id', selectedMessage.id);

        if (error) {
            errorMessage = error.message;
            return;
        }

        selectedMessage = null;
        await loadData();
        if (!errorMessage) notify('Pesan berhasil dihapus.');
    }

    async function logout() {
        const { error } = await supabase.auth.signOut();
        if (error) {
            errorMessage = error.message;
            return;
        }
        await goto('/login');
    }

    onMount(() => {
        init();
    });
</script>

<svelte:head>
    <title>Admin | LaundryKu</title>
    <meta name="description" content="Dashboard admin LaundryKu" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
    <link
        href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap"
        rel="stylesheet"
    />
</svelte:head>

<svelte:window
    onkeydown={(e) => {
        if (e.key === 'Escape') closeModals();
    }}
/>

{#if loading}
    <div class="loading-screen">
        <div class="loader"></div>
        <h3>Memuat dashboard admin</h3>
        <p>Memeriksa akun dan mengambil data pesanan...</p>
    </div>
{:else}
    <div class="layout">
        <!-- SIDEBAR -->
        <aside class="sidebar">
            <div class="brand">
                <div class="brand-icon">
                    <svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3c3 4 6 6.5 6 10a6 6 0 0 1-12 0c0-3.5 3-6 6-10Z" /></svg>
                </div>
                <div>
                    <h2>LaundryKu</h2>
                    <small>Panel admin</small>
                </div>
            </div>

            <nav class="nav" aria-label="Menu utama">
                <div class="menu-title">Menu</div>

                <button class:active={activeTab === 'pesanan'} aria-current={activeTab === 'pesanan' ? 'page' : undefined} onclick={() => setTab('pesanan')}>
                    <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><rect x="5" y="3" width="14" height="18" rx="3" /><path d="M9 8h6M9 12h6M9 16h3" /></svg>
                    <span class="nav-text">Pesanan</span>
                    {#if verifyCount > 0}<b class="hot">{verifyCount}</b>{:else}<b>{orders.length}</b>{/if}
                </button>

                <button class:active={activeTab === 'voucher'} aria-current={activeTab === 'voucher' ? 'page' : undefined} onclick={() => setTab('voucher')}>
                    <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M3 12V4h8l10 10-8 8L3 12Z" /><circle cx="7.5" cy="8.5" r="1.2" /></svg>
                    <span class="nav-text">Voucher</span>
                    <b>{vouchers.length}</b>
                </button>

                <button class:active={activeTab === 'ulasan'} aria-current={activeTab === 'ulasan' ? 'page' : undefined} onclick={() => setTab('ulasan')}>
                    <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 2.7 5.6 6.1.9-4.4 4.3 1 6.1L12 17l-5.4 2.9 1-6.1-4.4-4.3 6.1-.9L12 3Z" /></svg>
                    <span class="nav-text">Ulasan</span>
                    <b>{feedbacks.length}</b>
                </button>

                <button class:active={activeTab === 'pesan'} aria-current={activeTab === 'pesan' ? 'page' : undefined} onclick={() => setTab('pesan')}>
                    <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="5" width="18" height="14" rx="3" /><path d="m4 7 8 6 8-6" /></svg>
                    <span class="nav-text">Pesan</span>
                    {#if unreadCount > 0}<b class="hot">{unreadCount}</b>{/if}
                </button>
            </nav>

            <div class="sidebar-bottom">
                <div class="admin-info">
                    <div class="avatar">{initials(user?.email)}</div>
                    <div>
                        <strong>Administrator</strong>
                        <small>{user?.email}</small>
                    </div>
                </div>

                <button class="logout" onclick={logout} aria-label="Keluar">
                    <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 4H6a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h3M16 8l4 4-4 4M20 12H9" /></svg>
                    <span>Keluar</span>
                </button>
            </div>
        </aside>

        <!-- MAIN -->
        <main class="main">
            <header class="topbar">
                <div>
                    <p>LaundryKu / Admin</p>
                    <h1>{tabTitles[activeTab]}</h1>
                    <small>Kelola pesanan, pembayaran, promo, ulasan, dan pesan pelanggan.</small>
                </div>

                <div class="topbar-actions">
                    {#if activeTab === 'voucher'}
                        <button class="save-button" onclick={newVoucher}>+ Voucher baru</button>
                    {/if}
                    <button class="refresh" onclick={refreshData} disabled={refreshing}>
                        <svg class:spin={refreshing} viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 11a8 8 0 1 0-2.3 5.7M20 4v7h-7" /></svg>
                        <span>{refreshing ? 'Memuat...' : 'Refresh'}</span>
                    </button>
                </div>
            </header>

            <div aria-live="polite">
                {#if errorMessage}
                    <div class="alert error" role="alert">
                        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="M12 8v5M12 16h.01" /></svg>
                        <span>{errorMessage}</span>
                    </div>
                {/if}

                {#if warningMessage}
                    <div class="alert warn" role="status">
                        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="M12 8v5M12 16h.01" /></svg>
                        <span>{warningMessage}</span>
                    </div>
                {/if}

                {#if notice}
                    <div class="alert success" role="status">
                        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="m8 12 3 3 5-6" /></svg>
                        <span>{notice}</span>
                    </div>
                {/if}
            </div>

            <!-- ===================== PESANAN ===================== -->
            {#if activeTab === 'pesanan'}
                <section class="stats">
                    <div class="stat-card">
                        <div class="stat-icon blue"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 8 4.5v9L12 21l-8-4.5v-9L12 3Z" /><path d="m4 7.5 8 4.5 8-4.5M12 12v9" /></svg></div>
                        <div><span>Total pesanan</span><strong>{orders.length}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon orange"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="M12 7v5l3 2" /></svg></div>
                        <div><span>Menunggu</span><strong>{waitingCount}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon purple"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><rect x="4" y="3" width="16" height="18" rx="3" /><circle cx="12" cy="13" r="4" /><path d="M8 7h.01M11 7h.01" /></svg></div>
                        <div><span>Diproses</span><strong>{processingCount}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="m8 12 3 3 5-6" /></svg></div>
                        <div><span>Siap diambil</span><strong>{readyCount}</strong></div>
                    </div>
                    <button class="stat-card clickable" class:attention={verifyCount > 0} onclick={() => { paymentFilter = 'Menunggu Verifikasi'; statusFilter = 'Semua'; }}>
                        <div class="stat-icon red"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="6" width="18" height="12" rx="2" /><circle cx="12" cy="12" r="2.5" /></svg></div>
                        <div><span>Perlu verifikasi bayar</span><strong>{verifyCount}</strong></div>
                    </button>
                    <div class="stat-card">
                        <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3v18M16 7.5c0-1.7-1.8-3-4-3s-4 1.3-4 3 1.8 2.700 4 3 4 1.300 4 3-1.800 3-4 3-4-1.300-4-3" /></svg></div>
                        <div><span>Pendapatan (lunas)</span><strong class="money">{rupiah(revenue)}</strong></div>
                    </div>
                </section>

                <section class="panel">
                    <div class="panel-header">
                        <div>
                            <h2>Daftar semua pesanan</h2>
                            <p>Semua pesanan customer yang tersimpan di database.</p>
                        </div>
                    </div>

                    <div class="toolbar">
                        <div class="search">
                            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="7" /><path d="m20 20-3.5-3.5" /></svg>
                            <input bind:value={search} aria-label="Cari pesanan" placeholder="Cari nomor, nama, email, kode voucher..." />
                        </div>

                        <select bind:value={statusFilter} aria-label="Filter status pesanan">
                            <option value="Semua">Semua status</option>
                            {#each orderStatuses as status}<option value={status}>{status}</option>{/each}
                        </select>

                        <select bind:value={paymentFilter} aria-label="Filter status pembayaran">
                            <option value="Semua">Semua pembayaran</option>
                            {#each paymentStatuses as status}<option value={status}>{status}</option>{/each}
                        </select>
                    </div>

                    <div class="table-wrap">
                        <table>
                            <thead>
                                <tr>
                                    <th>Nomor pesanan</th>
                                    <th>Customer</th>
                                    <th>Layanan</th>
                                    <th>Jadwal</th>
                                    <th>Total</th>
                                    <th>Status</th>
                                    <th>Pembayaran</th>
                                    <th>Tanggal</th>
                                    <th class="th-action">Aksi</th>
                                </tr>
                            </thead>

                            <tbody>
                                {#each filteredOrders as order (order.id)}
                                    <tr>
                                        <td data-label="Nomor"><strong class="order-number">{order.order_number}</strong></td>

                                        <td data-label="Customer">
                                            <div class="cust">
                                                <strong>{customerFor(order)?.full_name || 'Customer'}</strong>
                                                <small class="sub-text">{customerFor(order)?.email || order.customer_id}</small>
                                            </div>
                                        </td>

                                        <td data-label="Layanan">
                                            <div class="cust">
                                                <strong>{order.service_name}</strong>
                                                <small class="sub-text">{order.weight_kg} kg</small>
                                            </div>
                                        </td>

                                        <td data-label="Jadwal">
                                            <div class="schedule">
                                                <span>{order.pickup ? `Jemput ${shortDay(order.pickup_date)}, ${order.pickup_time || '-'}` : 'Antar sendiri'}</span>
                                                <span>{order.delivery ? `Antar ${shortDay(order.delivery_date)}, ${order.delivery_time || '-'}` : 'Ambil sendiri'}</span>
                                            </div>
                                        </td>

                                        <td data-label="Total" class="price">
                                            {rupiah(order.total_amount)}
                                            {#if order.voucher_code}<small class="sub-text tag">🏷 {order.voucher_code}</small>{/if}
                                        </td>

                                        <td data-label="Status"><span class="status {statusClass(order.status)}">{order.status}</span></td>

                                        <td data-label="Pembayaran">
                                            <div class="pay-cell">
                                                <span class="status {payClass(order.payment_status)}">{order.payment_status}</span>
                                                <small class="sub-text">{methodLabel[order.payment_method] || '-'}</small>
                                            </div>
                                        </td>

                                        <td data-label="Tanggal" class="date">{dateFormat(order.created_at)}</td>

                                        <td class="cell-action">
                                            <button class="action-button" class:primary={order.payment_status === 'Menunggu Verifikasi'} onclick={() => editOrder(order)}>
                                                {order.payment_status === 'Menunggu Verifikasi' ? 'Verifikasi' : 'Atur'}
                                            </button>
                                        </td>
                                    </tr>
                                {:else}
                                    <tr class="empty-row">
                                        <td colspan="9" class="empty">
                                            <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M4 13 6.5 5h11L20 13M4 13v6h16v-6M4 13h4l1 2h6l1-2h4" /></svg></div>
                                            <strong>Tidak ada pesanan</strong>
                                            <p>Pesanan customer akan muncul di sini.</p>
                                        </td>
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>

                    <div class="table-footer">Menampilkan {filteredOrders.length} dari {orders.length} pesanan</div>
                </section>

            <!-- ===================== VOUCHER ===================== -->
            {:else if activeTab === 'voucher'}
                <section class="stats message-stats">
                    <div class="stat-card">
                        <div class="stat-icon blue"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M3 12V4h8l10 10-8 8L3 12Z" /><circle cx="7.5" cy="8.5" r="1.2" /></svg></div>
                        <div><span>Total voucher</span><strong>{vouchers.length}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="m8 12 3 3 5-6" /></svg></div>
                        <div><span>Aktif sekarang</span><strong>{activeVoucherCount}</strong></div>
                    </div>
                </section>

                <section class="panel">
                    <div class="panel-header">
                        <div>
                            <h2>Voucher & promo</h2>
                            <p>Voucher aktif otomatis tampil di halaman Promo customer.</p>
                        </div>
                    </div>

                    <div class="toolbar">
                        <div class="search">
                            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="7" /><path d="m20 20-3.5-3.5" /></svg>
                            <input bind:value={search} aria-label="Cari voucher" placeholder="Cari kode atau judul promo..." />
                        </div>
                    </div>

                    <div class="table-wrap">
                        <table class="msg-table">
                            <thead>
                                <tr>
                                    <th>Kode</th>
                                    <th>Promo</th>
                                    <th>Diskon</th>
                                    <th>Min. pesanan</th>
                                    <th>Berlaku s/d</th>
                                    <th>Dipakai</th>
                                    <th>Status</th>
                                    <th class="th-action">Aksi</th>
                                </tr>
                            </thead>
                            <tbody>
                                {#each filteredVouchers as v (v.id)}
                                    <tr>
                                        <td data-label="Kode"><span class="code-chip">{v.code}</span></td>
                                        <td data-label="Promo">
                                            <div class="cust">
                                                <strong>{v.title}</strong>
                                                <small class="sub-text wide">{v.description || '-'}</small>
                                            </div>
                                        </td>
                                        <td data-label="Diskon" class="price">
                                            {voucherValue(v)}
                                            {#if v.max_discount}<small class="sub-text">maks. {rupiah(v.max_discount)}</small>{/if}
                                        </td>
                                        <td data-label="Min. pesanan">{Number(v.min_order) > 0 ? rupiah(v.min_order) : '-'}</td>
                                        <td data-label="Berlaku s/d" class="date">{v.valid_until ? dayFormat(v.valid_until) : 'Tanpa batas'}</td>
                                        <td data-label="Dipakai">{orders.filter((o) => o.voucher_code === v.code && o.status !== 'Dibatalkan').length}×</td>
                                        <td data-label="Status">
                                            {#if isExpired(v)}
                                                <span class="status cancelled">Kedaluwarsa</span>
                                            {:else if v.is_active}
                                                <span class="status completed">Aktif</span>
                                            {:else}
                                                <span class="status waiting">Nonaktif</span>
                                            {/if}
                                        </td>
                                        <td class="cell-action">
                                            <div class="row-actions">
                                                <button class="action-button" onclick={() => toggleVoucher(v)}>{v.is_active ? 'Nonaktifkan' : 'Aktifkan'}</button>
                                                <button class="action-button" onclick={() => editVoucher(v)}>Ubah</button>
                                            </div>
                                        </td>
                                    </tr>
                                {:else}
                                    <tr class="empty-row">
                                        <td colspan="8" class="empty">
                                            <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M3 12V4h8l10 10-8 8L3 12Z" /><circle cx="7.5" cy="8.5" r="1.2" /></svg></div>
                                            <strong>Belum ada voucher</strong>
                                            <p>Klik “+ Voucher baru” untuk membuat promo pertama.</p>
                                        </td>
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>

                    <div class="table-footer">Menampilkan {filteredVouchers.length} dari {vouchers.length} voucher</div>
                </section>

            <!-- ===================== ULASAN ===================== -->
            {:else if activeTab === 'ulasan'}
                <section class="stats">
                    <div class="stat-card">
                        <div class="stat-icon orange"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 2.7 5.6 6.1.9-4.4 4.3 1 6.1L12 17l-5.4 2.9 1-6.1-4.4-4.3 6.1-.9L12 3Z" /></svg></div>
                        <div><span>Rating rata-rata</span><strong>{feedbacks.length ? avgRating.toFixed(1) : '-'}<small class="of"> / 5</small></strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon blue"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M4 5h16v11H9l-5 4V5Z" /></svg></div>
                        <div><span>Total ulasan</span><strong>{feedbacks.length}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3c3 4 6 6.5 6 10a6 6 0 0 1-12 0c0-3.5 3-6 6-10Z" /></svg></div>
                        <div><span>Kebersihan</span><strong>{aspectAvg.clean ? aspectAvg.clean.toFixed(1) : '-'}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon purple"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9" /><path d="M12 7v5l3 2" /></svg></div>
                        <div><span>Ketepatan waktu · Pelayanan</span><strong>{aspectAvg.time ? aspectAvg.time.toFixed(1) : '-'} · {aspectAvg.service ? aspectAvg.service.toFixed(1) : '-'}</strong></div>
                    </div>
                </section>

                <section class="panel">
                    <div class="panel-header">
                        <div>
                            <h2>Ulasan pelanggan</h2>
                            <p>Rating dan komentar dari pesanan yang sudah selesai.</p>
                        </div>
                    </div>

                    <div class="toolbar">
                        <div class="search">
                            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="7" /><path d="m20 20-3.5-3.5" /></svg>
                            <input bind:value={search} aria-label="Cari ulasan" placeholder="Cari nomor pesanan, nama, atau komentar..." />
                        </div>
                    </div>

                    <div class="table-wrap">
                        <table class="msg-table">
                            <thead>
                                <tr>
                                    <th>Pesanan</th>
                                    <th>Customer</th>
                                    <th>Rating</th>
                                    <th>Kebersihan</th>
                                    <th>Waktu</th>
                                    <th>Pelayanan</th>
                                    <th>Komentar</th>
                                    <th>Tanggal</th>
                                </tr>
                            </thead>
                            <tbody>
                                {#each filteredFeedbacks as f (f.id)}
                                    <tr>
                                        <td data-label="Pesanan"><strong class="order-number">{orderFor(f.order_id)?.order_number || '-'}</strong></td>
                                        <td data-label="Customer">
                                            <div class="cust">
                                                <strong>{profiles.find((p) => p.id === f.customer_id)?.full_name || 'Customer'}</strong>
                                                <small class="sub-text">{profiles.find((p) => p.id === f.customer_id)?.email || ''}</small>
                                            </div>
                                        </td>
                                        <td data-label="Rating"><span class="stars" aria-label="{f.rating} dari 5">{stars(f.rating)}</span></td>
                                        <td data-label="Kebersihan">{f.cleanliness ? stars(f.cleanliness) : '-'}</td>
                                        <td data-label="Waktu">{f.timeliness ? stars(f.timeliness) : '-'}</td>
                                        <td data-label="Pelayanan">{f.service ? stars(f.service) : '-'}</td>
                                        <td data-label="Komentar" class="comment">{f.comment || '-'}</td>
                                        <td data-label="Tanggal" class="date">{dateFormat(f.created_at)}</td>
                                    </tr>
                                {:else}
                                    <tr class="empty-row">
                                        <td colspan="8" class="empty">
                                            <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 2.7 5.6 6.1.9-4.4 4.3 1 6.1L12 17l-5.4 2.9 1-6.1-4.4-4.3 6.1-.9L12 3Z" /></svg></div>
                                            <strong>Belum ada ulasan</strong>
                                            <p>Ulasan akan muncul setelah pelanggan menilai pesanan yang selesai.</p>
                                        </td>
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>

                    <div class="table-footer">Menampilkan {filteredFeedbacks.length} dari {feedbacks.length} ulasan</div>
                </section>

            <!-- ===================== PESAN ===================== -->
            {:else}
                <section class="stats message-stats">
                    <div class="stat-card">
                        <div class="stat-icon blue"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="5" width="18" height="14" rx="3" /><path d="m4 7 8 6 8-6" /></svg></div>
                        <div><span>Total pesan</span><strong>{messages.length}</strong></div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-icon orange"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M6 17V11a6 6 0 0 1 12 0v6l2 2H4l2-2ZM10 21h4" /></svg></div>
                        <div><span>Belum dibaca</span><strong>{unreadCount}</strong></div>
                    </div>
                </section>

                <section class="panel">
                    <div class="panel-header">
                        <div>
                            <h2>Pesan dari customer</h2>
                            <p>Baca dan kelola pesan yang dikirim customer.</p>
                        </div>
                    </div>

                    <div class="toolbar">
                        <div class="search">
                            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="7" /><path d="m20 20-3.5-3.5" /></svg>
                            <input bind:value={search} aria-label="Cari pesan" placeholder="Cari nama, email, atau isi pesan..." />
                        </div>
                    </div>

                    <div class="table-wrap">
                        <table class="msg-table">
                            <thead>
                                <tr>
                                    <th>Nama</th><th>Email</th><th>Subjek</th><th>Pesan</th><th>Tanggal</th><th>Status</th><th class="th-action">Aksi</th>
                                </tr>
                            </thead>
                            <tbody>
                                {#each filteredMessages as message (message.id)}
                                    <tr class:unread={!message.is_read}>
                                        <td data-label="Nama"><strong>{message.name || 'Customer'}</strong></td>
                                        <td data-label="Email" class="email-cell">{message.email || '-'}</td>
                                        <td data-label="Subjek">{message.subject || '-'}</td>
                                        <td data-label="Pesan" class="preview"><span>{message.message}</span></td>
                                        <td data-label="Tanggal" class="date">{dateFormat(message.created_at)}</td>
                                        <td data-label="Status">
                                            {#if message.is_read}
                                                <span class="status completed">Sudah dibaca</span>
                                            {:else}
                                                <span class="status waiting">Belum dibaca</span>
                                            {/if}
                                        </td>
                                        <td class="cell-action"><button class="action-button" onclick={() => openMessage(message)}>Buka</button></td>
                                    </tr>
                                {:else}
                                    <tr class="empty-row">
                                        <td colspan="7" class="empty">
                                            <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="5" width="18" height="14" rx="3" /><path d="m4 7 8 6 8-6" /></svg></div>
                                            <strong>Belum ada pesan</strong>
                                            <p>Pesan dari customer akan muncul di sini.</p>
                                        </td>
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>

                    <div class="table-footer">Menampilkan {filteredMessages.length} dari {messages.length} pesan</div>
                </section>
            {/if}
        </main>
    </div>
{/if}

<!-- MODAL DETAIL, PEMBAYARAN & ATUR PESANAN -->
{#if selectedOrder}
    <div class="backdrop" role="presentation" onclick={(event) => { if (event.target === event.currentTarget) selectedOrder = null; }}>
        <section class="modal wide" role="dialog" aria-modal="true" aria-labelledby="order-title">
            <div class="modal-header">
                <div>
                    <span class="modal-label">Detail pesanan</span>
                    <h2 id="order-title">{selectedOrder.order_number}</h2>
                </div>
                <button class="close-button" aria-label="Tutup" onclick={() => (selectedOrder = null)}>
                    <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><path d="M6 6l12 12M18 6 6 18" /></svg>
                </button>
            </div>

            <div class="detail-box">
                <p><span>Nama customer</span><strong>{customerFor(selectedOrder)?.full_name || 'Customer'}</strong></p>
                <p><span>Email</span><strong>{customerFor(selectedOrder)?.email || '-'}</strong></p>
                <p><span>Layanan</span><strong>{selectedOrder.service_name}</strong></p>
                <p><span>Berat</span><strong>{selectedOrder.weight_kg} kg</strong></p>
                <p><span>Setrika / Express</span><strong>{selectedOrder.ironing ? 'Setrika' : 'Tanpa setrika'} · {selectedOrder.express ? 'Express' : 'Regular'}</strong></p>
                <p>
                    <span>Penjemputan</span>
                    <strong>
                        {#if selectedOrder.pickup}
                            {dayFormat(selectedOrder.pickup_date)}, {selectedOrder.pickup_time || '-'}<br />
                            <em>{selectedOrder.pickup_address || '-'}</em>
                        {:else}Diantar sendiri ke laundry{/if}
                    </strong>
                </p>
                <p>
                    <span>Pengantaran</span>
                    <strong>
                        {#if selectedOrder.delivery}
                            {dayFormat(selectedOrder.delivery_date)}, {selectedOrder.delivery_time || '-'}<br />
                            <em>{selectedOrder.delivery_address || '-'}</em>
                        {:else}Diambil sendiri di laundry{/if}
                    </strong>
                </p>
                <p><span>Catatan</span><strong>{selectedOrder.notes || '-'}</strong></p>
                <p><span>Tanggal pesanan</span><strong>{dateFormat(selectedOrder.created_at)}</strong></p>
                <p><span>Metode bayar</span><strong>{methodLabel[selectedOrder.payment_method] || '-'}</strong></p>
                {#if selectedOrder.voucher_code}
                    <p><span>Voucher</span><strong>{selectedOrder.voucher_code} (−{rupiah(selectedOrder.discount_amount)})</strong></p>
                {/if}
                {#if selectedOrder.points_used > 0}
                    <p><span>Poin ditukar</span><strong>{selectedOrder.points_used} poin (−{rupiah(selectedOrder.points_used * 50)})</strong></p>
                {/if}
                <p><span>Poin diperoleh</span><strong>+{selectedOrder.points_earned || 0} (masuk saat Selesai)</strong></p>
                {#if selectedOrder.paid_at}
                    <p><span>Dilunasi</span><strong>{dateFormat(selectedOrder.paid_at)}</strong></p>
                {/if}
                <p class="total-row"><span>Total pembayaran</span><strong>{rupiah(selectedOrder.total_amount)}</strong></p>
            </div>

            <!-- bukti pembayaran -->
            <div class="proof-box">
                <strong>Bukti pembayaran</strong>
                {#if !selectedOrder.payment_proof_url}
                    <p class="muted">{selectedOrder.payment_method === 'cod' ? 'Bayar di tempat — tidak ada bukti transfer.' : 'Customer belum mengunggah bukti.'}</p>
                {:else if proofLoading}
                    <p class="muted">Memuat bukti...</p>
                {:else if proofError}
                    <p class="muted err">{proofError}</p>
                {:else if proofUrl}
                    <a href={proofUrl} target="_blank" rel="noopener noreferrer" class="proof-link">
                        <img src={proofUrl} alt="Bukti pembayaran {selectedOrder.order_number}" />
                        <span>Buka ukuran penuh ↗</span>
                    </a>
                {/if}

                {#if selectedOrder.payment_status === 'Menunggu Verifikasi'}
                    <div class="quick-actions">
                        <button class="save-button" onclick={() => (selectedOrder.payment_status = 'Lunas')}>✓ Tandai Lunas</button>
                        <button class="delete-button" onclick={() => (selectedOrder.payment_status = 'Ditolak')}>Tolak bukti</button>
                    </div>
                {/if}
            </div>

            <div class="form-grid">
                <div class="form-group">
                    <label for="order-status">Status pesanan</label>
                    <select id="order-status" bind:value={selectedOrder.status}>
                        {#each orderStatuses as status}<option value={status}>{status}</option>{/each}
                    </select>
                </div>

                <div class="form-group">
                    <label for="payment-status">Status pembayaran</label>
                    <select id="payment-status" bind:value={selectedOrder.payment_status}>
                        {#each paymentStatuses as status}<option value={status}>{status}</option>{/each}
                    </select>
                </div>
            </div>

            {#if selectedOrder.status === 'Selesai' && selectedOrder.payment_status !== 'Lunas'}
                <p class="hint">
                    Pesanan ditandai Selesai tetapi pembayaran belum Lunas.
                    {selectedOrder.payment_status === 'Bayar di Tempat' ? 'Ubah ke Lunas setelah uang diterima.' : ''}
                </p>
            {/if}

            <div class="modal-actions">
                <button class="cancel-button" onclick={() => (selectedOrder = null)} disabled={saving}>Batal</button>
                <button class="save-button" onclick={saveOrder} disabled={saving}>{saving ? 'Menyimpan...' : 'Simpan perubahan'}</button>
            </div>
        </section>
    </div>
{/if}

<!-- MODAL VOUCHER -->
{#if voucherForm}
    <div class="backdrop" role="presentation" onclick={(event) => { if (event.target === event.currentTarget) voucherForm = null; }}>
        <section class="modal" role="dialog" aria-modal="true" aria-labelledby="voucher-title">
            <div class="modal-header">
                <div>
                    <span class="modal-label">Voucher & promo</span>
                    <h2 id="voucher-title">{voucherForm.id ? 'Ubah voucher' : 'Voucher baru'}</h2>
                </div>
                <button class="close-button" aria-label="Tutup" onclick={() => (voucherForm = null)}>
                    <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><path d="M6 6l12 12M18 6 6 18" /></svg>
                </button>
            </div>

            {#if errorMessage}<div class="alert error" role="alert"><span>{errorMessage}</span></div>{/if}

            <div class="form-group first">
                <label for="v-code">Kode voucher</label>
                <input id="v-code" class="upper" bind:value={voucherForm.code} placeholder="HEMAT10" maxlength="24" />
            </div>
            <div class="form-group">
                <label for="v-title">Judul promo</label>
                <input id="v-title" bind:value={voucherForm.title} placeholder="Diskon 10% semua layanan" />
            </div>
            <div class="form-group">
                <label for="v-desc">Deskripsi (opsional)</label>
                <input id="v-desc" bind:value={voucherForm.description} placeholder="Syarat singkat promo" />
            </div>

            <div class="form-grid">
                <div class="form-group">
                    <label for="v-type">Jenis diskon</label>
                    <select id="v-type" bind:value={voucherForm.discount_type}>
                        <option value="percent">Persen (%)</option>
                        <option value="fixed">Nominal (Rp)</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="v-value">Nilai {voucherForm.discount_type === 'percent' ? '(%)' : '(Rp)'}</label>
                    <input id="v-value" type="number" min="1" inputmode="numeric" bind:value={voucherForm.discount_value} />
                </div>
                <div class="form-group">
                    <label for="v-min">Minimal pesanan (Rp)</label>
                    <input id="v-min" type="number" min="0" inputmode="numeric" bind:value={voucherForm.min_order} />
                </div>
                <div class="form-group">
                    <label for="v-max">Maks. diskon (Rp)</label>
                    <input id="v-max" type="number" min="0" inputmode="numeric" bind:value={voucherForm.max_discount} placeholder="Tanpa batas" disabled={voucherForm.discount_type === 'fixed'} />
                </div>
                <div class="form-group">
                    <label for="v-until">Berlaku sampai</label>
                    <input id="v-until" type="date" bind:value={voucherForm.valid_until} />
                </div>
                <div class="form-group">
                    <label for="v-active">Status</label>
                    <select id="v-active" bind:value={voucherForm.is_active}>
                        <option value={true}>Aktif</option>
                        <option value={false}>Nonaktif</option>
                    </select>
                </div>
            </div>

            <div class="modal-actions">
                {#if voucherForm.id}
                    <button class="delete-button" onclick={() => deleteVoucher(voucherForm)} disabled={saving}>Hapus</button>
                {/if}
                <button class="cancel-button" onclick={() => (voucherForm = null)} disabled={saving}>Batal</button>
                <button class="save-button" onclick={saveVoucher} disabled={saving}>{saving ? 'Menyimpan...' : 'Simpan voucher'}</button>
            </div>
        </section>
    </div>
{/if}

<!-- MODAL PESAN CUSTOMER -->
{#if selectedMessage}
    <div class="backdrop" role="presentation" onclick={(event) => { if (event.target === event.currentTarget) selectedMessage = null; }}>
        <section class="modal" role="dialog" aria-modal="true" aria-labelledby="msg-title">
            <div class="modal-header">
                <div>
                    <span class="modal-label">Pesan customer</span>
                    <h2 id="msg-title">Detail pesan</h2>
                </div>
                <button class="close-button" aria-label="Tutup" onclick={() => (selectedMessage = null)}>
                    <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><path d="M6 6l12 12M18 6 6 18" /></svg>
                </button>
            </div>

            <div class="detail-box">
                <p><span>Nama</span><strong>{selectedMessage.name || '-'}</strong></p>
                <p><span>Email</span><strong>{selectedMessage.email || '-'}</strong></p>
                <p><span>Subjek</span><strong>{selectedMessage.subject || '-'}</strong></p>
                <p><span>Tanggal</span><strong>{dateFormat(selectedMessage.created_at)}</strong></p>
            </div>

            <div class="message-content">
                <strong>Isi pesan</strong>
                <p>{selectedMessage.message}</p>
            </div>

            <div class="modal-actions">
                <button class="delete-button" onclick={deleteMessage}>Hapus pesan</button>

                {#if selectedMessage.email}
                    <a class="save-button email-button" href="mailto:{selectedMessage.email}?subject=Re%3A%20{encodeURIComponent(selectedMessage.subject || 'Pesan LaundryKu')}">Balas email</a>
                {/if}

                <button class="cancel-button" onclick={() => (selectedMessage = null)}>Tutup</button>
            </div>
        </section>
    </div>
{/if}

<style>
    :global(*) { box-sizing: border-box; }

    :global(body) {
        margin: 0;
        background: #f1f6f9;
        color: #0c2b45;
        font-family: 'Plus Jakarta Sans', Inter, 'Segoe UI', Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
    }

    :global(:root) {
        --ink: #0c2b45;
        --muted: #5b7186;
        --soft: #8aa0b2;
        --line: #e1eaf0;
        --teal: #0e7c86;
        --teal-dark: #0a6670;
        --aqua: #2bc4c0;
        --foam: #e7f6f6;
        --gold: #f5a524;
    }

    button, input, select { font: inherit; }
    button { cursor: pointer; }
    button:disabled { opacity: 0.6; cursor: not-allowed; }

    button:focus-visible, input:focus-visible, select:focus-visible, a:focus-visible {
        outline: 3px solid rgba(14, 124, 134, 0.4);
        outline-offset: 2px;
    }

    /* ---------- loading ---------- */
    .loading-screen {
        min-height: 100vh; min-height: 100dvh;
        display: flex; flex-direction: column; justify-content: center; align-items: center;
        text-align: center; padding: 20px;
    }
    .loading-screen h3 { margin: 18px 0 6px; }
    .loading-screen p { color: var(--muted); margin: 0; }
    .loader {
        width: 44px; height: 44px; border-radius: 50%;
        border: 4px solid #cfe9ea; border-top-color: var(--teal);
        animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
    .spin { animation: spin 0.8s linear infinite; }

    .layout { min-height: 100vh; min-height: 100dvh; }

    /* ---------- sidebar ---------- */
    .sidebar {
        position: fixed; inset: 0 auto 0 0; width: 264px; z-index: 5;
        display: flex; flex-direction: column;
        padding: 24px 16px;
        color: #e3f1f6;
        background:
            radial-gradient(360px 240px at 100% 0, rgba(43, 196, 192, 0.22), transparent 70%),
            linear-gradient(170deg, #0c2b45, #082238);
    }

    .brand { display: flex; align-items: center; gap: 12px; padding: 4px 10px 28px; }
    .brand-icon {
        width: 40px; height: 40px; border-radius: 12px; display: grid; place-items: center;
        color: #fff; background: linear-gradient(145deg, var(--aqua), var(--teal));
        box-shadow: 0 8px 20px rgba(43, 196, 192, 0.3);
    }
    .brand h2 { margin: 0; font-size: 19px; font-weight: 800; letter-spacing: -0.3px; color: #fff; }
    .brand small { color: #86a8b8; font-size: 12px; }

    .menu-title { color: #6f94a6; font-size: 12px; font-weight: 600; margin: 4px 12px 10px; }

    .nav { display: flex; flex-direction: column; gap: 4px; }
    .nav button {
        display: flex; align-items: center; gap: 12px; width: 100%;
        border: 0; background: transparent; color: #b9d2de;
        padding: 12px; border-radius: 12px; text-align: left;
        font-size: 14px; font-weight: 600;
        transition: background 0.18s, color 0.18s;
    }
    .nav button:hover { background: rgba(255, 255, 255, 0.07); color: #fff; }
    .nav button.active { background: rgba(43, 196, 192, 0.16); color: #fff; box-shadow: inset 3px 0 0 var(--aqua); }
    .nav button.active svg { color: var(--aqua); }
    .nav b {
        margin-left: auto; min-width: 24px; text-align: center;
        background: rgba(255, 255, 255, 0.12); color: #e3f1f6;
        padding: 3px 8px; border-radius: 20px; font-size: 11px;
    }
    .nav b.hot { background: #ff6b5e; color: #fff; }

    .sidebar-bottom { margin-top: auto; }
    .admin-info {
        display: flex; gap: 11px; align-items: center;
        padding: 16px 6px; border-top: 1px solid rgba(255, 255, 255, 0.1);
    }
    .avatar {
        flex: none; width: 38px; height: 38px; border-radius: 12px; display: grid; place-items: center;
        background: rgba(43, 196, 192, 0.2); color: var(--aqua); font-weight: 800;
    }
    .admin-info strong, .admin-info small {
        display: block; max-width: 160px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
    }
    .admin-info strong { font-size: 13px; color: #fff; }
    .admin-info small { color: #86a8b8; font-size: 12px; margin-top: 3px; }

    .sidebar .logout {
        width: 100%; display: flex; align-items: center; gap: 10px;
        border: 0; border-radius: 12px; padding: 12px;
        color: #ffb3ab; background: rgba(255, 107, 94, 0.12);
        font-weight: 700; font-size: 14px; transition: background 0.18s;
    }
    .sidebar .logout:hover { background: rgba(255, 107, 94, 0.22); }

    /* ---------- main ---------- */
    .main { margin-left: 264px; padding: 32px clamp(20px, 3vw, 40px); max-width: 1700px; }

    .topbar { display: flex; justify-content: space-between; align-items: flex-end; gap: 16px; margin-bottom: 26px; }
    .topbar p { color: var(--soft); font-size: 13px; font-weight: 600; margin: 0 0 8px; }
    .topbar h1 { font-size: 28px; letter-spacing: -0.8px; margin: 0 0 6px; font-weight: 800; }
    .topbar small { color: var(--muted); font-size: 14px; }
    .topbar-actions { display: flex; gap: 10px; align-items: center; }

    .refresh {
        display: inline-flex; align-items: center; gap: 8px;
        background: #fff; border: 1.5px solid var(--line); color: var(--ink);
        padding: 11px 16px; border-radius: 12px; font-weight: 700; font-size: 14px; white-space: nowrap;
        transition: border-color 0.18s, box-shadow 0.18s;
    }
    .refresh:hover:not(:disabled) { border-color: var(--teal); box-shadow: 0 4px 14px rgba(14, 124, 134, 0.12); }

    .alert {
        display: flex; gap: 10px; align-items: flex-start;
        padding: 13px 16px; border-radius: 12px; margin-bottom: 18px; font-size: 14px; line-height: 1.5;
        animation: pop 0.25s ease;
    }
    .alert svg { flex: none; margin-top: 1px; }
    .error { color: #b42318; background: #fef3f2; border: 1px solid #fcd0cc; }
    .success { color: #0b7a4b; background: #eafaf2; border: 1px solid #b5ead0; }
    .warn { color: #8a4b08; background: #fff7e8; border: 1px solid #f6dcaa; }
    @keyframes pop { from { opacity: 0; transform: translateY(-4px); } to { opacity: 1; transform: none; } }

    /* ---------- stats ---------- */
    .stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 16px; margin-bottom: 22px; }
    .message-stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }

    .stat-card {
        display: flex; align-items: center; gap: 14px; text-align: left; color: inherit;
        background: #fff; border: 1px solid var(--line); border-radius: 16px; padding: 18px 20px;
    }
    .stat-card.clickable { transition: box-shadow 0.18s, transform 0.18s, border-color 0.18s; }
    .stat-card.clickable:hover { box-shadow: 0 10px 24px rgba(12, 43, 69, 0.08); transform: translateY(-1px); }
    .stat-card.attention { border-color: #f5b8b2; background: #fffafa; }
    .stat-icon { flex: none; width: 46px; height: 46px; border-radius: 13px; display: grid; place-items: center; }
    .blue { background: #e8f1fd; color: #2563c9; }
    .orange { background: #fff2e2; color: #c2620a; }
    .purple { background: #efeafd; color: #6a45d1; }
    .green { background: #e3f7ee; color: #0b8a55; }
    .red { background: #fef0ee; color: #b42318; }
    .stat-card span { display: block; color: var(--muted); font-size: 13px; margin-bottom: 4px; font-weight: 500; }
    .stat-card strong { font-size: 26px; letter-spacing: -0.5px; font-weight: 800; }
    .stat-card strong.money { font-size: 19px; }
    .stat-card .of { font-size: 14px; color: var(--soft); font-weight: 600; letter-spacing: 0; }

    /* ---------- panel & table ---------- */
    .panel { background: #fff; border: 1px solid var(--line); border-radius: 18px; overflow: hidden; }
    .panel-header { padding: 22px 24px 4px; }
    .panel-header h2 { margin: 0 0 6px; font-size: 18px; letter-spacing: -0.3px; }
    .panel-header p { margin: 0; color: var(--muted); font-size: 13.5px; }

    .toolbar { display: flex; gap: 12px; padding: 18px 24px; flex-wrap: wrap; }
    .search { position: relative; flex: 1; min-width: 220px; }
    .search svg { position: absolute; left: 14px; top: 50%; transform: translateY(-50%); color: var(--soft); pointer-events: none; }
    .search input { padding-left: 42px; width: 100%; }

    input, select {
        min-height: 44px; padding: 10px 14px;
        border: 1.5px solid var(--line); background: #fbfdfe; color: var(--ink);
        border-radius: 12px; outline: none; font-size: 15px;
        transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
    }
    input::placeholder { color: #9fb1c0; }
    input:hover, select:hover { border-color: #c3d4e0; }
    input:focus, select:focus { border-color: var(--teal); background: #fff; box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
    input:disabled { background: #f1f5f8; }
    input.upper { text-transform: uppercase; }

    .table-wrap { width: 100%; overflow-x: auto; }
    table { width: 100%; min-width: 1180px; border-collapse: collapse; }
    .msg-table { min-width: 940px; }

    th, td { padding: 15px 14px; text-align: left; border-bottom: 1px solid #edf2f6; font-size: 13.5px; vertical-align: middle; }
    th:first-child, td:first-child { padding-left: 24px; }
    th:last-child, td:last-child { padding-right: 24px; }
    th {
        background: #f6fafc; color: var(--muted); font-size: 12px; font-weight: 700; white-space: nowrap;
        position: sticky; top: 0;
    }
    td { color: #3d566d; }
    tbody tr { transition: background 0.15s; }
    tbody tr:hover { background: #f8fcfc; }
    tr.unread td:first-child { box-shadow: inset 3px 0 0 var(--aqua); }
    tr.unread strong { color: var(--ink); }

    .order-number { color: var(--teal); white-space: nowrap; }
    .cust strong { color: var(--ink); }
    .sub-text { display: block; max-width: 200px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--soft); font-size: 12px; margin-top: 3px; font-weight: 500; }
    .sub-text.wide { max-width: 280px; }
    .sub-text.tag { color: var(--teal); font-weight: 700; }
    .price { font-weight: 800; color: var(--ink); white-space: nowrap; }
    .date { white-space: nowrap; color: var(--muted); }
    .schedule { display: grid; gap: 3px; font-size: 12.5px; color: var(--muted); white-space: nowrap; }
    .pay-cell { display: grid; gap: 4px; justify-items: start; }
    .stars { color: var(--gold); letter-spacing: 1px; white-space: nowrap; }
    .comment { max-width: 280px; line-height: 1.55; overflow-wrap: anywhere; }
    .code-chip { display: inline-block; padding: 5px 11px; border-radius: 8px; background: var(--foam); color: var(--teal-dark); font-size: 12.5px; font-weight: 800; letter-spacing: 0.5px; border: 1px dashed #9fd3d6; }

    .status {
        display: inline-flex; align-items: center; gap: 7px;
        padding: 5px 11px; border-radius: 99px; font-size: 12px; font-weight: 700; white-space: nowrap;
    }
    .status::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: currentColor; }
    .waiting { background: #fff2e2; color: #b45a09; }
    .processing { background: #e8f1fd; color: #1f56b8; }
    .ready { background: #dff5f4; color: #0a7a7f; }
    .completed { background: #e3f7ee; color: #0b7a4b; }
    .cancelled { background: #fef0ee; color: #b42318; }

    .th-action { text-align: right; }
    .cell-action { text-align: right; }
    .row-actions { display: flex; gap: 6px; justify-content: flex-end; flex-wrap: wrap; }
    .action-button {
        border: 1.5px solid #cfe3e6; background: #fff; color: var(--teal-dark);
        padding: 8px 16px; border-radius: 10px; font-weight: 700; font-size: 13px; white-space: nowrap;
        transition: all 0.18s;
    }
    .action-button:hover { background: var(--teal); border-color: var(--teal); color: #fff; }
    .action-button.primary { background: var(--teal); border-color: var(--teal); color: #fff; }
    .action-button.primary:hover { background: var(--teal-dark); }

    .preview { max-width: 240px; }
    .preview span { display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

    .empty { text-align: center; padding: 56px 16px; color: var(--soft); }
    tr.empty-row:hover { background: transparent; }
    .empty-ico {
        width: 60px; height: 60px; margin: 0 auto 14px; border-radius: 18px;
        display: grid; place-items: center; background: var(--foam); color: var(--teal);
    }
    .empty strong { display: block; color: var(--ink); font-size: 15px; }
    .empty p { margin: 6px 0 0; font-size: 13.5px; }

    .table-footer { padding: 14px 24px; color: var(--soft); font-size: 13px; }

    /* ---------- modal ---------- */
    .backdrop {
        position: fixed; inset: 0; z-index: 20; display: grid; place-items: center;
        padding: 18px; overflow-y: auto;
        background: rgba(8, 34, 56, 0.55); backdrop-filter: blur(3px);
        animation: fade 0.2s ease;
    }
    @keyframes fade { from { opacity: 0; } to { opacity: 1; } }
    @keyframes rise { from { opacity: 0; transform: translateY(14px) scale(0.98); } to { opacity: 1; transform: none; } }

    .modal {
        background: #fff; width: min(580px, 100%); max-height: 92vh; max-height: 92dvh; overflow-y: auto;
        border-radius: 20px; padding: 24px; box-shadow: 0 30px 80px rgba(8, 34, 56, 0.35);
        animation: rise 0.25s ease;
    }
    .modal.wide { width: min(640px, 100%); }
    .modal-header { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 18px; }
    .modal-label { font-size: 13px; color: var(--teal); font-weight: 700; }
    .modal h2 { margin: 6px 0 0; font-size: 22px; letter-spacing: -0.4px; }
    .close-button {
        width: 36px; height: 36px; border: 0; border-radius: 10px; display: grid; place-items: center;
        background: #eef4f8; color: var(--muted); transition: background 0.18s; flex: none;
    }
    .close-button:hover { background: #dfe9f0; }

    .detail-box { background: #f6fafc; border: 1px solid #e8f0f5; border-radius: 14px; padding: 6px 16px; }
    .detail-box p {
        display: flex; justify-content: space-between; gap: 16px;
        border-bottom: 1px solid #e8f0f5; padding: 11px 0; margin: 0; font-size: 13.5px;
    }
    .detail-box p:last-child { border-bottom: 0; }
    .detail-box span { color: var(--muted); flex: none; }
    .detail-box strong { text-align: right; overflow-wrap: anywhere; max-width: 65%; }
    .detail-box em { font-style: normal; color: var(--muted); font-weight: 500; font-size: 12.5px; }
    .total-row strong { color: var(--teal); font-size: 16px; }

    .proof-box { margin-top: 16px; padding: 16px; border: 1.5px dashed #cfdde6; border-radius: 14px; }
    .proof-box > strong { font-size: 13px; }
    .proof-box .muted { color: var(--muted); font-size: 13.5px; margin: 8px 0 0; }
    .proof-box .err { color: #b42318; }
    .proof-link { display: block; margin-top: 10px; text-decoration: none; color: var(--teal-dark); font-weight: 700; font-size: 13px; }
    .proof-link img { display: block; width: 100%; max-height: 300px; object-fit: contain; background: #f6fafc; border-radius: 12px; border: 1px solid var(--line); margin-bottom: 8px; }
    .quick-actions { display: flex; gap: 10px; margin-top: 14px; flex-wrap: wrap; }
    .quick-actions .delete-button { margin-right: 0; }

    .hint { margin: 14px 0 0; padding: 11px 14px; border-radius: 12px; background: #fff7e8; border: 1px solid #f6dcaa; color: #8a4b08; font-size: 13px; line-height: 1.5; }

    .form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 0 14px; }
    .form-group { margin-top: 18px; }
    .form-group.first { margin-top: 0; }
    .form-group label { display: block; font-size: 13px; font-weight: 700; margin-bottom: 8px; color: #243f57; }
    .form-group select, .form-group input { width: 100%; }

    .modal-actions { display: flex; justify-content: flex-end; flex-wrap: wrap; gap: 10px; margin-top: 24px; }
    .cancel-button, .save-button, .delete-button {
        border: 0; border-radius: 12px; padding: 12px 18px; font-size: 14px; font-weight: 700; text-decoration: none;
        transition: transform 0.15s, filter 0.15s, background 0.15s;
    }
    .cancel-button { background: #eef4f8; color: #3d566d; }
    .cancel-button:hover:not(:disabled) { background: #e1ebf1; }
    .save-button { background: linear-gradient(135deg, var(--teal), #0b5f78); color: #fff; box-shadow: 0 8px 18px rgba(14, 124, 134, 0.25); white-space: nowrap; }
    .save-button:hover:not(:disabled) { filter: brightness(1.06); transform: translateY(-1px); }
    .delete-button { background: #fef0ee; color: #b42318; margin-right: auto; }
    .delete-button:hover:not(:disabled) { background: #fde0dc; }
    .email-button { display: inline-flex; align-items: center; }

    .message-content { background: #f6fafc; border: 1px solid #e8f0f5; padding: 16px; margin-top: 16px; border-radius: 14px; }
    .message-content strong { font-size: 13px; }
    .message-content p { font-size: 14.5px; line-height: 1.75; white-space: pre-wrap; overflow-wrap: anywhere; margin: 10px 0 0; color: #3d566d; }

    /* ================= RESPONSIVE ================= */
    @media (max-width: 1100px) {
        .stats { grid-template-columns: repeat(2, minmax(0, 1fr)); }
    }

    /* Tablet & HP: sidebar jadi header atas */
    @media (max-width: 900px) {
        .sidebar {
            position: sticky; top: 0; inset: auto; width: 100%;
            flex-direction: row; flex-wrap: wrap; align-items: center; justify-content: space-between;
            padding: 12px 16px 10px; gap: 10px;
            padding-top: calc(12px + env(safe-area-inset-top));
        }
        .brand { padding: 0; }
        .brand-icon { width: 36px; height: 36px; }
        .brand small { display: none; }
        .menu-title, .admin-info { display: none; }
        .sidebar-bottom { margin: 0; order: 2; }
        .sidebar .logout { width: 42px; height: 42px; padding: 0; justify-content: center; }
        .sidebar .logout span { display: none; }

        .nav { order: 3; flex-direction: row; width: 100%; background: rgba(255, 255, 255, 0.08); padding: 4px; border-radius: 14px; overflow-x: auto; }
        .nav button { flex: 1 0 auto; width: auto; justify-content: center; padding: 10px; gap: 8px; }
        .nav button.active { box-shadow: none; background: #fff; color: var(--teal-dark); }
        .nav button.active svg { color: var(--teal); }
        .nav b { margin-left: 0; }

        .main { margin: 0; padding: 22px 18px 32px; }
    }

    @media (max-width: 560px) {
        .nav-text { display: none; }
    }

    /* HP: tabel jadi kartu */
    @media (max-width: 760px) {
        .topbar { align-items: flex-start; }
        .topbar h1 { font-size: 23px; }
        .topbar small { display: none; }
        .refresh { padding: 10px 12px; }
        .refresh span { display: none; }

        .panel { background: transparent; border: 0; border-radius: 0; overflow: visible; }
        .panel-header { padding: 0 2px 2px; }
        .toolbar { padding: 14px 0; flex-direction: column; }
        .toolbar select { width: 100%; }
        .search input { background: #fff; }

        .table-wrap { overflow: visible; }
        table, .msg-table { min-width: 0; display: block; }
        thead { display: none; }
        tbody { display: grid; gap: 12px; }

        tbody tr:not(.empty-row) {
            display: block; background: #fff; border: 1px solid var(--line);
            border-radius: 16px; padding: 6px 16px 14px;
        }
        tbody tr:hover { background: #fff; }
        tr.unread { box-shadow: inset 4px 0 0 var(--aqua); }
        tr.unread td:first-child { box-shadow: none; }

        td, td:first-child, td:last-child {
            display: flex; justify-content: space-between; align-items: center; gap: 16px;
            padding: 10px 0; border-bottom: 1px dashed #e5edf2; text-align: right;
        }
        td[data-label]::before {
            content: attr(data-label); flex: none; color: var(--soft); font-size: 12px; font-weight: 600; text-align: left;
        }
        td .cust, .pay-cell { text-align: right; justify-items: end; }
        .schedule { text-align: right; white-space: normal; }
        .sub-text, .sub-text.wide { max-width: 190px; margin-left: auto; }
        .preview { max-width: none; }
        .preview span { max-width: 200px; }
        .email-cell, .comment { overflow-wrap: anywhere; }
        .cell-action { border-bottom: 0; padding: 14px 0 0; display: block; text-align: right; }
        .cell-action .action-button { width: 100%; padding: 11px; }
        .row-actions > .action-button { flex: 1; }

        tr.empty-row { display: block; background: #fff; border: 1px solid var(--line); border-radius: 16px; }
        tr.empty-row td { display: block; text-align: center; border: 0; }

        .table-footer { padding: 14px 2px 0; }

        .stats { gap: 10px; }
        .stat-card { padding: 14px; gap: 10px; flex-direction: column; align-items: flex-start; }
        .stat-icon { width: 40px; height: 40px; }
        .stat-card strong { font-size: 22px; }
        .stat-card strong.money { font-size: 16px; }

        .backdrop { align-items: end; padding: 0; }
        .modal, .modal.wide { width: 100%; max-height: 94dvh; border-radius: 22px 22px 0 0; padding: 20px 18px calc(20px + env(safe-area-inset-bottom)); animation: sheet 0.28s ease; }
        .form-grid { grid-template-columns: 1fr; }
        .modal-actions { flex-direction: column-reverse; }
        .modal-actions > *, .quick-actions > * { width: 100%; justify-content: center; text-align: center; margin: 0; }
        .quick-actions { flex-direction: column; }
        .topbar-actions .save-button { padding: 10px 12px; }
    }
    @keyframes sheet { from { transform: translateY(40px); opacity: 0; } to { transform: none; opacity: 1; } }

    @media (max-width: 380px) {
        .stat-card strong { font-size: 20px; }
        .main { padding: 18px 14px 28px; }
    }

    @media (prefers-reduced-motion: reduce) {
        * { animation: none !important; transition: none !important; }
    }
</style>