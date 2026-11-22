<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';

  // ---------- util tanggal (dideklarasikan lebih dulu karena dipakai state awal) ----------
  function toISO(d) { return new Date(d.getTime() - d.getTimezoneOffset() * 60000).toISOString().slice(0, 10); }
  function addDays(iso, n) { const d = iso ? new Date(iso + 'T00:00:00') : new Date(); d.setDate(d.getDate() + n); return toISO(d); }
  const today = toISO(new Date());

  let activePage = 'dashboard';
  let showOrderModal = false;
  let showNotifications = false;
  let searchQuery = '';
  let loading = true;
  let savingOrder = false;
  let pageError = '';
  let currentUser = null;
  let customer = { name: '', email: '', initials: '' };
  let orders = [];
  let vouchers = [];
  let feedbacks = {}; // key: uuid pesanan
  let toast = null;
  let toastTimer;

  // modal lain
  let showDetail = null;
  let showPay = null;
  let showReview = null;

  // pembayaran
  let payMethod = 'qris';
  let proofFile = null;
  let proofPreview = '';
  let paying = false;

  // review
  let review = { overall: 0, clean: 0, time: 0, service: 0, comment: '' };
  let savingReview = false;

  // voucher di form pesanan
  let voucherInput = '';
  let appliedVoucher = null;
  let voucherMsg = '';
  let voucherOk = false;
  let checkingVoucher = false;
  let usePoints = false;

  function newForm() {
    return {
      weight: 3, ironing: false, express: false,
      pickup: false, address: '', pickupDate: today, pickupTime: '', notes: '', photo: null, photoName: '',
      delivery: false, deliverySame: true, deliveryAddress: '', deliveryDate: '', deliveryTime: '',
      paymentMethod: 'qris'
    };
  }
  let form = newForm();

  // ---------- konfigurasi harga & loyalty ----------
  const pricePerKg = 5000;
  const ironingPrice = 2000;
  const expressPrice = 3000;
  const pickupPrice = 5000;
  const deliveryPrice = 5000;
  const earnPerRupiah = 1000; // 1 poin tiap Rp1.000 (dihitung dari total bayar)
  const pointValue = 50;      // 1 poin = Rp50 saat ditukar

  const tiers = [
    { name: 'Bronze', min: 0 },
    { name: 'Silver', min: 200 },
    { name: 'Gold', min: 500 },
    { name: 'Platinum', min: 1000 }
  ];
  const rewards = [
    { points: 100, value: 5000 },
    { points: 200, value: 10000 },
    { points: 500, value: 25000 }
  ];

  const timeSlots = ['08:00 – 10:00', '10:00 – 12:00', '13:00 – 15:00', '15:00 – 17:00', '17:00 – 19:00'];

  // ---------- konfigurasi pembayaran (GANTI dengan data toko kamu) ----------
  const qrisImage = ''; // contoh: '/qris-laundryku.png' (taruh gambar di folder static)
  const bankAccounts = [
    { bank: 'BCA', number: '1234567890', name: 'LaundryKu' },
    { bank: 'BRI', number: '0987654321', name: 'LaundryKu' }
  ];
  const paymentMethods = [
    { id: 'qris', label: 'QRIS', hint: 'Scan via e-wallet / m-banking', icon: 'qr' },
    { id: 'transfer', label: 'Transfer bank', hint: 'BCA / BRI', icon: 'bank' },
    { id: 'cod', label: 'Bayar di tempat', hint: 'Saat laundry selesai', icon: 'cash' }
  ];
  const methodLabel = { qris: 'QRIS', transfer: 'Transfer bank', cod: 'Bayar di tempat' };

  const ratingFields = [
    { key: 'overall', label: 'Kepuasan keseluruhan', big: true },
    { key: 'clean', label: 'Kebersihan' },
    { key: 'time', label: 'Ketepatan waktu' },
    { key: 'service', label: 'Pelayanan' }
  ];

  const navItems = [
    { id: 'dashboard', label: 'Dashboard', short: 'Beranda', icon: 'home' },
    { id: 'orders', label: 'Pesanan Saya', short: 'Pesanan', icon: 'orders' },
    { id: 'services', label: 'Layanan Laundry', short: 'Layanan', icon: 'drop' },
    { id: 'promo', label: 'Promo & Voucher', short: 'Promo', icon: 'tag' },
    { id: 'loyalty', label: 'Poin Loyalty', short: 'Poin', icon: 'gift' },
    { id: 'profile', label: 'Profil Saya', short: 'Profil', icon: 'user' }
  ];

  // Ikon SVG statis (stroke), dipakai lewat {@html}
  const icons = {
    home: '<path d="M4 11 12 4l8 7v8a1 1 0 0 1-1 1h-4v-6H9v6H5a1 1 0 0 1-1-1v-8Z"/>',
    orders: '<rect x="5" y="3" width="14" height="18" rx="3"/><path d="M9 8h6M9 12h6M9 16h3"/>',
    drop: '<path d="M12 3c3 4 6 6.5 6 10a6 6 0 0 1-12 0c0-3.5 3-6 6-10Z"/>',
    user: '<circle cx="12" cy="8" r="4"/><path d="M4 21c1-4 4-6 8-6s7 2 8 6"/>',
    bell: '<path d="M6 17V11a6 6 0 0 1 12 0v6l2 2H4l2-2ZM10 21h4"/>',
    logout: '<path d="M9 4H6a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h3M16 8l4 4-4 4M20 12H9"/>',
    plus: '<path d="M12 5v14M5 12h14"/>',
    clock: '<circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 2"/>',
    check: '<circle cx="12" cy="12" r="9"/><path d="m8 12 3 3 5-6"/>',
    scale: '<path d="M7 8h10l3 12H4L7 8Z"/><circle cx="12" cy="5" r="2"/>',
    search: '<circle cx="11" cy="11" r="7"/><path d="m20 20-3.5-3.5"/>',
    bolt: '<path d="M13 3 5 14h6l-1 7 8-11h-6l1-7Z"/>',
    iron: '<path d="M3 17h18l-2-7a4 4 0 0 0-4-3H8a5 5 0 0 0-5 5v5Z"/><path d="M8 12h5"/>',
    truck: '<path d="M3 7h11v9H3zM14 10h4l3 3v3h-7"/><circle cx="7" cy="18" r="1.8"/><circle cx="17" cy="18" r="1.8"/>',
    upload: '<path d="M12 16V5M7 10l5-5 5 5M5 20h14"/>',
    info: '<circle cx="12" cy="12" r="9"/><path d="M12 11v5M12 8h.01"/>',
    arrow: '<path d="M5 12h14M13 6l6 6-6 6"/>',
    close: '<path d="M6 6l12 12M18 6 6 18"/>',
    box: '<path d="m12 3 8 4.5v9L12 21l-8-4.5v-9L12 3Z"/><path d="m4 7.5 8 4.5 8-4.5M12 12v9"/>',
    calendar: '<rect x="4" y="5" width="16" height="15" rx="3"/><path d="M8 3v4M16 3v4M4 10h16"/>',
    home2: '<path d="M4 11 12 4l8 7v8a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1v-8Z"/>',
    tag: '<path d="M3 12V4h8l10 10-8 8L3 12Z"/><circle cx="7.5" cy="8.5" r="1.2"/>',
    gift: '<rect x="3" y="9" width="18" height="11" rx="2"/><path d="M12 9v11M3 13h18M12 9c-2-4-6-3-5 0 .5 1 2.5 1 5 0Zm0 0c2-4 6-3 5 0-.5 1-2.5 1-5 0Z"/>',
    star: '<path d="m12 3 2.7 5.6 6.1.9-4.4 4.3 1 6.1L12 17l-5.4 2.9 1-6.1-4.4-4.3 6.1-.9L12 3Z"/>',
    receipt: '<path d="M6 3h12v18l-3-2-3 2-3-2-3 2V3Z"/><path d="M9 8h6M9 12h6"/>',
    qr: '<rect x="4" y="4" width="6" height="6" rx="1"/><rect x="14" y="4" width="6" height="6" rx="1"/><rect x="4" y="14" width="6" height="6" rx="1"/><path d="M14 14h2v2h-2zM18 14h2M14 18h2M18 18h2v2"/>',
    bank: '<path d="M3 10 12 4l9 6M5 10v8M9 10v8M15 10v8M19 10v8M3 20h18"/>',
    cash: '<rect x="3" y="6" width="18" height="12" rx="2"/><circle cx="12" cy="12" r="2.5"/>',
    copy: '<rect x="9" y="9" width="11" height="11" rx="2"/><path d="M5 15V6a2 2 0 0 1 2-2h9"/>',
    print: '<path d="M7 9V3h10v6M7 17H5a2 2 0 0 1-2-2v-4a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2v4a2 2 0 0 1-2 2h-2"/><rect x="7" y="14" width="10" height="7" rx="1"/>'
  };

  // ---------- perhitungan harga (reaktif) ----------
  $: weightNum = Number(form.weight || 0);
  $: washAmount = weightNum * pricePerKg;
  $: ironAmount = form.ironing ? weightNum * ironingPrice : 0;
  $: expressAmount = form.express ? weightNum * expressPrice : 0;
  $: subtotal = washAmount + ironAmount + expressAmount; // layanan saja (dasar voucher)
  $: pickupFee = form.pickup ? pickupPrice : 0;
  $: deliveryFee = form.delivery ? deliveryPrice : 0;
  $: gross = subtotal + pickupFee + deliveryFee;
  $: voucherDiscount = calcDiscount(appliedVoucher, subtotal);
  $: afterVoucher = gross - voucherDiscount;
  $: maxPoints = Math.max(0, Math.min(pointsBalance, Math.floor((afterVoucher * 0.5) / pointValue)));
  $: pointsUsed = usePoints ? maxPoints : 0;
  $: pointsDiscount = pointsUsed * pointValue;
  $: total = Math.max(0, afterVoucher - pointsDiscount);
  $: pointsEarnEstimate = Math.floor(total / earnPerRupiah);
  $: minDelivery = addDays(form.pickup && form.pickupDate ? form.pickupDate : today, form.express ? 1 : 3);

  // ---------- turunan data pesanan ----------
  $: filteredOrders = orders.filter(order => order.id.toLowerCase().includes(searchQuery.toLowerCase()) || order.service.toLowerCase().includes(searchQuery.toLowerCase()));
  $: completedOrders = orders.filter(o => o.status === 'Selesai').length;
  $: activeOrders = orders.filter(o => o.status !== 'Selesai' && o.status !== 'Dibatalkan').length;
  $: totalWeight = orders.reduce((sum, o) => sum + o.weight, 0);
  $: pageTitle = navItems.find(n => n.id === activePage)?.label || 'Dashboard';

  $: pointsEarned = orders.filter(o => o.status === 'Selesai').reduce((s, o) => s + o.pointsEarned, 0);
  $: pointsSpent = orders.filter(o => o.status !== 'Dibatalkan').reduce((s, o) => s + o.pointsUsed, 0);
  $: pointsBalance = Math.max(0, pointsEarned - pointsSpent);
  $: pointsPending = orders.filter(o => o.status !== 'Selesai' && o.status !== 'Dibatalkan').reduce((s, o) => s + o.pointsEarned, 0);
  $: currentTier = [...tiers].reverse().find(t => pointsEarned >= t.min) || tiers[0];
  $: nextTier = tiers.find(t => t.min > pointsEarned);
  $: tierProgress = nextTier ? Math.min(100, Math.round(((pointsEarned - currentTier.min) / (nextTier.min - currentTier.min)) * 100)) : 100;
  $: pointHistory = orders.filter(o => o.pointsEarned > 0 || o.pointsUsed > 0);

  $: unpaidOrders = orders.filter(o => o.paymentStatus === 'Belum Dibayar' && o.status !== 'Dibatalkan');
  $: toReview = orders.filter(o => o.status === 'Selesai' && !feedbacks[o.uuid]);
  $: notifCount = unpaidOrders.length + toReview.length;

  // ---------- helper ----------
  function showToast(text, type = 'success') {
    toast = { text, type };
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => (toast = null), 4000);
  }
  function formatCurrency(value) {
    return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(value || 0);
  }
  function formatDate(value) {
    return new Date(value).toLocaleDateString('id-ID', { day: 'numeric', month: 'short', year: 'numeric' });
  }
  function formatDay(value) { return value ? formatDate(value + 'T12:00:00') : '-'; }
  function initials(name = '') {
    return name.trim().split(/\s+/).slice(0, 2).map(part => part[0]?.toUpperCase() || '').join('') || 'CU';
  }
  function calcDiscount(v, base) {
    if (!v || base <= 0 || base < Number(v.min_order || 0)) return 0;
    let d = v.discount_type === 'percent' ? (base * Number(v.discount_value)) / 100 : Number(v.discount_value);
    if (v.max_discount) d = Math.min(d, Number(v.max_discount));
    return Math.round(Math.min(d, base));
  }
  function voucherValueLabel(v) {
    return v.discount_type === 'percent' ? `${Number(v.discount_value)}%` : formatCurrency(v.discount_value);
  }
  function paymentColorOf(status) {
    return status === 'Lunas' ? 'completed'
      : status === 'Menunggu Verifikasi' ? 'processing'
      : status === 'Bayar di Tempat' ? 'ready'
      : status === 'Ditolak' ? 'cancelled' : 'pending';
  }
  function mapOrder(row) {
    const service = row.service_name || (row.ironing ? 'Wash & Iron' : 'Wash Only') + (row.express ? ' Express' : '');
    const statusColor =
      row.status === 'Selesai' ? 'completed' :
      row.status === 'Diproses' ? 'processing' :
      row.status === 'Siap Diambil' ? 'ready' :
      row.status === 'Dibatalkan' ? 'cancelled' : 'pending';
    const paymentStatus = row.payment_status || 'Belum Dibayar';
    return {
      uuid: row.id,
      id: row.order_number || row.id,
      date: formatDate(row.created_at),
      service,
      weight: Number(row.weight_kg),
      total: Number(row.total_amount),
      status: row.status || 'Menunggu',
      statusColor,
      ironing: !!row.ironing,
      express: !!row.express,
      pickup: !!row.pickup,
      address: row.pickup_address || '',
      pickupDate: row.pickup_date || '',
      pickupTime: row.pickup_time || '',
      delivery: !!row.delivery,
      deliveryAddress: row.delivery_address || '',
      deliveryDate: row.delivery_date || '',
      deliveryTime: row.delivery_time || '',
      notes: row.notes || '',
      paymentMethod: row.payment_method || '',
      paymentStatus,
      paymentColor: paymentColorOf(paymentStatus),
      proofUrl: row.payment_proof_url || '',
      voucherCode: row.voucher_code || '',
      voucherDiscount: Number(row.discount_amount || 0),
      pointsUsed: Number(row.points_used || 0),
      pointsEarned: Number(row.points_earned || 0)
    };
  }
  function receiptLines(o) {
    const w = o.weight;
    const lines = [{ label: `Cuci (${w} kg × ${formatCurrency(pricePerKg)})`, value: w * pricePerKg }];
    if (o.ironing) lines.push({ label: 'Setrika', value: w * ironingPrice });
    if (o.express) lines.push({ label: 'Express', value: w * expressPrice });
    if (o.pickup) lines.push({ label: 'Pickup', value: pickupPrice });
    if (o.delivery) lines.push({ label: 'Pengantaran', value: deliveryPrice });
    if (o.voucherDiscount) lines.push({ label: `Voucher ${o.voucherCode}`, value: -o.voucherDiscount });
    if (o.pointsUsed) lines.push({ label: `Tukar ${o.pointsUsed} poin`, value: -o.pointsUsed * pointValue });
    return lines;
  }

  // ---------- load data ----------
  async function loadDashboard() {
    loading = true; pageError = '';
    const { data: { user }, error: authError } = await supabase.auth.getUser();
    if (authError || !user) { await goto('/login'); return; }
    currentUser = user;

    const { data: profile, error: profileError } = await supabase.from('profiles').select('full_name,email').eq('id', user.id).maybeSingle();
    if (profileError) console.warn('Profile load:', profileError.message);
    const displayName = profile?.full_name || user.user_metadata?.full_name || user.user_metadata?.name || user.email?.split('@')[0] || 'Customer';
    customer = { name: displayName, email: profile?.email || user.email || '', initials: initials(displayName) };

    const { data: rows, error: ordersError } = await supabase.from('customer_orders').select('*').eq('customer_id', user.id).order('created_at', { ascending: false });
    if (ordersError) pageError = 'Data pesanan gagal dimuat: ' + ordersError.message;
    orders = (rows || []).map(mapOrder);

    // voucher & feedback tidak boleh membuat halaman gagal dimuat bila tabelnya belum ada
    const { data: vRows, error: vErr } = await supabase.from('vouchers').select('*').eq('is_active', true);
    if (vErr) console.warn('Voucher load:', vErr.message);
    vouchers = (vErr ? [] : vRows || []).filter(v => !v.valid_until || v.valid_until >= today);

    const { data: fRows, error: fErr } = await supabase.from('order_feedback').select('*').eq('customer_id', user.id);
    if (fErr) console.warn('Feedback load:', fErr.message);
    feedbacks = Object.fromEntries((fRows || []).map(f => [f.order_id, f]));

    loading = false;
  }

  onMount(() => {
    loadDashboard();
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      if (!session) goto('/login');
    });
    return () => subscription.unsubscribe();
  });

  // ---------- form pesanan ----------
  function handlePhoto(event) {
    const file = event.target.files?.[0];
    if (!file) return;
    if (file.size > 5 * 1024 * 1024) { showToast('Ukuran foto maksimal 5 MB.', 'error'); return; }
    form.photo = URL.createObjectURL(file);
    form.photoName = file.name;
  }
  function resetForm() {
    if (form.photo) URL.revokeObjectURL(form.photo);
    form = newForm();
    appliedVoucher = null; voucherInput = ''; voucherMsg = ''; voucherOk = false; usePoints = false;
  }

  async function applyVoucher(code = voucherInput) {
    const c = (code || '').trim().toUpperCase();
    if (!c) { voucherOk = false; voucherMsg = 'Masukkan kode voucher.'; return; }
    checkingVoucher = true; voucherMsg = '';
    let v = vouchers.find(x => x.code.toUpperCase() === c);
    if (!v) {
      const { data, error } = await supabase.from('vouchers').select('*').eq('code', c).eq('is_active', true).maybeSingle();
      if (!error) v = data;
    }
    checkingVoucher = false;
    if (!v || (v.valid_until && v.valid_until < today)) {
      appliedVoucher = null; voucherOk = false; voucherMsg = 'Kode voucher tidak valid atau sudah kedaluwarsa.'; return;
    }
    if (subtotal < Number(v.min_order || 0)) {
      appliedVoucher = null; voucherOk = false; voucherMsg = `Voucher ini berlaku untuk pesanan minimal ${formatCurrency(v.min_order)}.`; return;
    }
    appliedVoucher = v; voucherInput = c; voucherOk = true;
    voucherMsg = `Voucher ${c} terpasang — hemat ${formatCurrency(calcDiscount(v, subtotal))}.`;
  }
  function removeVoucher() { appliedVoucher = null; voucherInput = ''; voucherMsg = ''; voucherOk = false; }
  function usePromo(v) {
    showOrderModal = true;
    voucherInput = v.code;
    applyVoucher(v.code);
  }

  async function submitOrder() {
    if (!currentUser) { await goto('/login'); return; }
    if (!form.weight || Number(form.weight) <= 0) { showToast('Masukkan berat pakaian yang valid.', 'error'); return; }
    if (form.pickup) {
      if (!form.address.trim()) { showToast('Masukkan alamat penjemputan terlebih dahulu.', 'error'); return; }
      if (!form.pickupDate || form.pickupDate < today) { showToast('Pilih tanggal penjemputan (hari ini atau setelahnya).', 'error'); return; }
      if (!form.pickupTime) { showToast('Pilih jam penjemputan.', 'error'); return; }
    }
    const deliveryAddr = form.deliverySame && form.pickup ? form.address.trim() : form.deliveryAddress.trim();
    if (form.delivery) {
      if (!deliveryAddr) { showToast('Masukkan alamat pengantaran.', 'error'); return; }
      if (!form.deliveryDate || form.deliveryDate < minDelivery) { showToast(`Tanggal pengantaran paling cepat ${formatDay(minDelivery)}.`, 'error'); return; }
      if (!form.deliveryTime) { showToast('Pilih jam pengantaran.', 'error'); return; }
    }
    savingOrder = true;
    const serviceName = form.ironing ? `Wash & Iron${form.express ? ' Express' : ''}` : `Wash Only${form.express ? ' Express' : ''}`;
    const payload = {
      customer_id: currentUser.id,
      service_name: serviceName,
      weight_kg: Number(form.weight),
      ironing: form.ironing,
      express: form.express,
      pickup: form.pickup,
      pickup_address: form.pickup ? form.address.trim() : null,
      pickup_date: form.pickup ? form.pickupDate : null,
      pickup_time: form.pickup ? form.pickupTime : null,
      notes: form.pickup ? form.notes.trim() : null,
      delivery: form.delivery,
      delivery_address: form.delivery ? deliveryAddr : null,
      delivery_date: form.delivery ? form.deliveryDate : null,
      delivery_time: form.delivery ? form.deliveryTime : null,
      payment_method: form.paymentMethod,
      payment_status: form.paymentMethod === 'cod' ? 'Bayar di Tempat' : 'Belum Dibayar',
      voucher_code: appliedVoucher && voucherDiscount > 0 ? appliedVoucher.code : null,
      discount_amount: voucherDiscount,
      points_used: pointsUsed,
      points_earned: pointsEarnEstimate,
      total_amount: total,
    };
    const { data, error } = await supabase.from('customer_orders').insert(payload).select().single();
    savingOrder = false;
    if (error) { showToast('Pesanan gagal disimpan: ' + error.message, 'error'); return; }
    const created = mapOrder(data);
    orders = [created, ...orders];
    const method = form.paymentMethod;
    showOrderModal = false;
    activePage = 'orders';
    resetForm();
    if (method !== 'cod' && created.total > 0) {
      openPay(created, method);
      showToast('Pesanan tersimpan. Selesaikan pembayaran ya!');
    } else {
      showToast('Pesanan berhasil disimpan!');
    }
  }

  // ---------- pembayaran ----------
  function openPay(order, method) {
    showDetail = null;
    payMethod = method || (order.paymentMethod && order.paymentMethod !== 'cod' ? order.paymentMethod : 'qris');
    proofFile = null; proofPreview = '';
    showPay = order;
  }
  function handleProof(event) {
    const file = event.target.files?.[0];
    if (!file) return;
    if (file.size > 5 * 1024 * 1024) { showToast('Ukuran bukti maksimal 5 MB.', 'error'); return; }
    proofFile = file;
    proofPreview = URL.createObjectURL(file);
  }
  async function copyText(text) {
    try { await navigator.clipboard.writeText(text); showToast('Nomor rekening disalin.'); }
    catch { showToast('Gagal menyalin, salin manual ya.', 'error'); }
  }
  async function confirmPay() {
    const order = showPay;
    if (!order) return;
    if (payMethod !== 'cod' && !proofFile && !order.proofUrl) { showToast('Unggah bukti pembayaran terlebih dahulu.', 'error'); return; }
    paying = true;
    let proofPath = order.proofUrl || null;
    if (payMethod !== 'cod' && proofFile) {
      const ext = (proofFile.name.split('.').pop() || 'jpg').toLowerCase();
      const path = `${currentUser.id}/${order.uuid}-${Date.now()}.${ext}`;
      const { error: upErr } = await supabase.storage.from('payment-proofs').upload(path, proofFile, { upsert: true });
      if (upErr) { paying = false; showToast('Bukti gagal diunggah: ' + upErr.message, 'error'); return; }
      proofPath = path;
    }
    const status = payMethod === 'cod' ? 'Bayar di Tempat' : 'Menunggu Verifikasi';
    const { data, error } = await supabase.from('customer_orders')
      .update({ payment_method: payMethod, payment_status: status, payment_proof_url: proofPath })
      .eq('id', order.uuid).select().single();
    paying = false;
    if (error) { showToast('Gagal menyimpan pembayaran: ' + error.message, 'error'); return; }
    const updated = mapOrder(data);
    orders = orders.map(o => (o.uuid === updated.uuid ? updated : o));
    if (proofPreview) URL.revokeObjectURL(proofPreview);
    proofFile = null; proofPreview = '';
    showPay = null;
    showDetail = updated;
    showToast(payMethod === 'cod' ? 'Metode bayar di tempat disimpan.' : 'Bukti terkirim. Admin akan memverifikasi pembayaranmu.');
  }

  // ---------- review ----------
  function openReview(order) {
    showDetail = null;
    const f = feedbacks[order.uuid];
    review = f
      ? { overall: f.rating, clean: f.cleanliness || 0, time: f.timeliness || 0, service: f.service || 0, comment: f.comment || '' }
      : { overall: 0, clean: 0, time: 0, service: 0, comment: '' };
    showReview = order;
  }
  async function submitReview() {
    if (!review.overall) { showToast('Pilih rating bintang terlebih dahulu.', 'error'); return; }
    savingReview = true;
    const row = {
      order_id: showReview.uuid,
      customer_id: currentUser.id,
      rating: review.overall,
      cleanliness: review.clean || null,
      timeliness: review.time || null,
      service: review.service || null,
      comment: review.comment.trim() || null
    };
    const { data, error } = await supabase.from('order_feedback').upsert(row, { onConflict: 'order_id' }).select().single();
    savingReview = false;
    if (error) { showToast('Ulasan gagal disimpan: ' + error.message, 'error'); return; }
    feedbacks = { ...feedbacks, [data.order_id]: data };
    showReview = null;
    showToast('Terima kasih atas ulasanmu!');
  }

  async function logout() {
    const { error } = await supabase.auth.signOut();
    if (error) showToast(error.message, 'error');
    else await goto('/login');
  }
  function closeAll() { showOrderModal = false; showNotifications = false; showDetail = null; showPay = null; showReview = null; }
  function navigate(page) { activePage = page; closeAll(); }
</script>

<svelte:head>
  <title>Dashboard | LaundryKu</title>
  <meta name="description" content="Dashboard pelanggan LaundryKu" />
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet" />
</svelte:head>

<svelte:window onkeydown={(e) => { if (e.key === 'Escape') closeAll(); }} />

<!-- baris tabel pesanan (dipakai di Dashboard & Pesanan Saya) -->
{#snippet orderRows(list)}
  {#each list as order (order.uuid)}
    <div class="table-row">
      <div class="order-id" data-label="ID">{order.id}</div>
      <div class="muted" data-label="Tanggal">{order.date}</div>
      <div class="service-cell" data-label="Layanan">
        <div class="service-icon"><svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg></div>
        {order.service}
      </div>
      <div data-label="Berat">{order.weight} kg</div>
      <div class="price-cell" data-label="Total">{formatCurrency(order.total)}</div>
      <div data-label="Status">
        <div class="badge-stack">
          <span class="status-badge {order.statusColor}"><span class="status-dot"></span>{order.status}</span>
          {#if order.status !== 'Dibatalkan'}
            <span class="status-badge sm {order.paymentColor}">{order.paymentStatus}</span>
          {/if}
        </div>
      </div>
      <div data-label="Aksi">
        <div class="row-actions">
          <button class="small-btn" onclick={() => (showDetail = order)}>Detail</button>
          {#if order.paymentStatus === 'Belum Dibayar' && order.status !== 'Dibatalkan'}
            <button class="small-btn solid" onclick={() => openPay(order)}>Bayar</button>
          {:else if order.status === 'Selesai'}
            <button class="small-btn solid" onclick={() => openReview(order)}>{feedbacks[order.uuid] ? 'Ulasanku' : 'Beri ulasan'}</button>
          {/if}
        </div>
      </div>
    </div>
  {/each}
{/snippet}

<div class="app-shell">
  <!-- SIDEBAR (desktop) -->
  <aside class="sidebar">
    <div class="brand">
      <div class="brand-icon">
        <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg>
      </div>
      <div class="brand-text">
        <strong>LaundryKu</strong>
        <small>Portal customer</small>
      </div>
    </div>

    <div class="nav-label">Menu</div>

    <nav class="navigation" aria-label="Menu utama">
      {#each navItems as item}
        <button class:active={activePage === item.id} aria-current={activePage === item.id ? 'page' : undefined} onclick={() => navigate(item.id)}>
          <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons[item.icon]}</svg>
          <span>{item.label}</span>
          {#if item.id === 'orders'}<span class="nav-count">{orders.length}</span>{/if}
          {#if item.id === 'loyalty'}<span class="nav-count">{pointsBalance}</span>{/if}
          {#if item.id === 'promo' && vouchers.length}<span class="nav-count">{vouchers.length}</span>{/if}
        </button>
      {/each}
    </nav>

    <div class="sidebar-promo">
      <strong>Pakaian bersih,<br />hari lebih ringan.</strong>
      <p>Pesan dalam hitungan menit, kami urus sisanya.</p>
      <button onclick={() => (showOrderModal = true)}>
        Pesan sekarang
        <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.arrow}</svg>
      </button>
    </div>

    <div class="sidebar-bottom">
      <div class="sidebar-user">
        <div class="avatar">{customer.initials}</div>
        <div class="user-info">
          <strong>{customer.name}</strong>
          <small>{currentTier.name} · {pointsBalance} poin</small>
        </div>
        <button class="icon-btn" title="Keluar" aria-label="Keluar" onclick={logout}>
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.logout}</svg>
        </button>
      </div>
    </div>
  </aside>

  <!-- MAIN -->
  <main class="main-content">
    <header class="topbar">
      <div class="mobile-brand">
        <div class="brand-icon sm">
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg>
        </div>
        <strong>LaundryKu</strong>
      </div>

      <div class="breadcrumb">
        <span>Workspace</span>
        <span class="crumb-separator">/</span>
        <strong>{pageTitle}</strong>
      </div>

      <div class="topbar-actions">
        <div class="date-chip">
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.calendar}</svg>
          {new Date().toLocaleDateString('id-ID', { day: 'numeric', month: 'long', year: 'numeric' })}
        </div>

        <button class="icon-btn bordered" aria-label="Notifikasi" aria-expanded={showNotifications} onclick={() => (showNotifications = !showNotifications)}>
          <svg viewBox="0 0 24 24" width="19" height="19" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.bell}</svg>
          {#if notifCount > 0}<span class="notification-dot"></span>{/if}
        </button>

        <button class="icon-btn bordered mobile-only" aria-label="Keluar" onclick={logout}>
          <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.logout}</svg>
        </button>

        {#if showNotifications}
          <div class="notification-popup">
            <strong>Notifikasi</strong>
            {#if notifCount === 0}
              <p>Tidak ada notifikasi baru.</p>
            {/if}
            {#if unpaidOrders.length}
              <p>{unpaidOrders.length} pesanan belum dibayar. <button class="link-btn" onclick={() => openPay(unpaidOrders[0])}>Bayar sekarang</button></p>
            {/if}
            {#if toReview.length}
              <p>{toReview.length} pesanan selesai menunggu ulasanmu. <button class="link-btn" onclick={() => openReview(toReview[0])}>Beri ulasan</button></p>
            {/if}
          </div>
        {/if}
      </div>
    </header>

    {#if loading}
      <div class="page-content">
        <div class="state-card">
          <div class="loader"></div>
          <h3>Memuat data akun...</h3>
          <p>Menyiapkan profil dan pesanan kamu.</p>
        </div>
      </div>
    {:else if pageError}
      <div class="page-content">
        <div class="message error" role="alert">{pageError}</div>
        <button class="primary-button" onclick={loadDashboard}>Coba lagi</button>
      </div>
    {:else if activePage === 'dashboard'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Ringkasan akun</div>
            <h1>Halo, {customer.name || 'Customer'}</h1>
            <p>Semua pakaian bersihmu, dalam satu tempat.</p>
          </div>
          <button class="primary-button" onclick={() => (showOrderModal = true)}>
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.plus}</svg>
            Pesan laundry
          </button>
        </section>

        <section class="stats-grid">
          <div class="stat-card">
            <div class="stat-icon blue"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.box}</svg></div>
            <div><span class="stat-label">Total pesanan</span><h2>{orders.length}</h2></div>
          </div>
          <div class="stat-card">
            <div class="stat-icon orange"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.clock}</svg></div>
            <div><span class="stat-label">Pesanan aktif</span><h2>{activeOrders}</h2></div>
          </div>
          <div class="stat-card">
            <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.check}</svg></div>
            <div><span class="stat-label">Pesanan selesai</span><h2>{completedOrders}</h2></div>
          </div>
          <div class="stat-card">
            <div class="stat-icon purple"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.scale}</svg></div>
            <div><span class="stat-label">Total berat</span><h2>{totalWeight} <small>kg</small></h2></div>
          </div>
        </section>

        <!-- RINGKASAN LOYALTY & TINDAKAN -->
        <section class="action-row">
          <button class="loyalty-mini" onclick={() => navigate('loyalty')}>
            <div class="stat-icon purple"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.gift}</svg></div>
            <div class="lm-text">
              <span class="stat-label">Poin loyalty · {currentTier.name}</span>
              <strong>{pointsBalance} poin <small>≈ {formatCurrency(pointsBalance * pointValue)}</small></strong>
              <div class="progress"><i style="width:{tierProgress}%"></i></div>
              <small class="lm-hint">{nextTier ? `${nextTier.min - pointsEarned} poin lagi menuju ${nextTier.name}` : 'Level tertinggi tercapai 🎉'}</small>
            </div>
          </button>

          <div class="todo-card">
            <strong>Perlu tindakan</strong>
            {#if notifCount === 0}
              <p>Semua beres. Tidak ada yang perlu kamu lakukan.</p>
            {/if}
            {#if unpaidOrders.length}
              <div class="todo-item">
                <span><b>{unpaidOrders.length}</b> pesanan belum dibayar</span>
                <button class="small-btn solid" onclick={() => openPay(unpaidOrders[0])}>Bayar</button>
              </div>
            {/if}
            {#if toReview.length}
              <div class="todo-item">
                <span><b>{toReview.length}</b> pesanan menunggu ulasan</span>
                <button class="small-btn solid" onclick={() => openReview(toReview[0])}>Ulas</button>
              </div>
            {/if}
            {#if vouchers.length}
              <div class="todo-item">
                <span><b>{vouchers.length}</b> promo tersedia</span>
                <button class="small-btn" onclick={() => navigate('promo')}>Lihat</button>
              </div>
            {/if}
          </div>
        </section>

        <!-- HERO -->
        <section class="hero-banner">
          <div class="hero-bg"></div>
          <div class="hero-content">
            <div class="hero-tag">Layanan laundry premium</div>
            <h2>Lebih banyak waktu<br />untuk hal yang berarti.</h2>
            <p>Biarkan kami merawat pakaianmu. Layanan praktis dengan hasil bersih, wangi, dan rapi.</p>
            <button class="hero-button" onclick={() => (showOrderModal = true)}>
              Mulai pesanan
              <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.arrow}</svg>
            </button>
          </div>

          <div class="hero-decoration" aria-hidden="true">
            <div class="drum">
              <div class="water"><i></i><i></i></div>
              <span class="bub b1"></span><span class="bub b2"></span><span class="bub b3"></span>
            </div>
            <div class="floating-card">
              <div class="floating-check">
                <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6 9 17l-5-5"/></svg>
              </div>
              <div><strong>Fresh &amp; clean</strong><small>Dirawat sepenuh hati</small></div>
            </div>
          </div>
        </section>

        <!-- RECENT ORDERS -->
        <section class="orders-section">
          <div class="section-heading">
            <div>
              <h2>Pesanan terbaru</h2>
              <p>Pantau status laundry kamu di sini.</p>
            </div>
            <button class="text-button" onclick={() => navigate('orders')}>
              Lihat semua
              <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.arrow}</svg>
            </button>
          </div>

          <div class="table-card">
            <div class="table-header">
              <span>ID pesanan</span><span>Tanggal</span><span>Layanan</span><span>Berat</span><span>Total</span><span>Status</span><span>Aksi</span>
            </div>

            {@render orderRows(orders.slice(0, 4))}

            {#if orders.length === 0}
              <div class="empty-state">
                <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg></div>
                <h3>Belum ada pesanan</h3>
                <p>Pesanan yang kamu buat akan muncul di sini.</p>
                <button class="primary-button" onclick={() => (showOrderModal = true)}>Buat pesanan pertama</button>
              </div>
            {/if}
          </div>
        </section>
      </div>

    {:else if activePage === 'orders'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Kelola pesanan</div>
            <h1>Pesanan saya</h1>
            <p>Kelola dan pantau semua pesanan laundry kamu.</p>
          </div>
          <button class="primary-button" onclick={() => (showOrderModal = true)}>
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.plus}</svg>
            Pesan laundry
          </button>
        </section>

        <div class="orders-toolbar">
          <div class="search-box">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.search}</svg>
            <input aria-label="Cari pesanan" placeholder="Cari ID pesanan atau layanan..." bind:value={searchQuery} />
          </div>
        </div>

        <div class="table-card">
          <div class="table-header">
            <span>ID pesanan</span><span>Tanggal</span><span>Layanan</span><span>Berat</span><span>Total</span><span>Status</span><span>Aksi</span>
          </div>

          {@render orderRows(filteredOrders)}

          {#if filteredOrders.length === 0}
            <div class="empty-state">
              <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">{@html icons.search}</svg></div>
              <h3>{orders.length === 0 ? 'Belum ada pesanan' : 'Pesanan tidak ditemukan'}</h3>
              <p>{orders.length === 0 ? 'Pesanan yang kamu buat akan muncul di sini.' : 'Coba kata pencarian yang berbeda.'}</p>
            </div>
          {/if}
        </div>
      </div>

    {:else if activePage === 'services'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Layanan kami</div>
            <h1>Layanan laundry</h1>
            <p>Pilih layanan sesuai kebutuhan pakaianmu.</p>
          </div>
        </section>

        <div class="services-grid">
          <div class="service-card">
            <div class="large-service-icon blue-bg"><svg viewBox="0 0 24 24" width="24" height="24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg></div>
            <div class="service-tag">Populer</div>
            <h3>Wash Only</h3>
            <p>Cuci bersih dengan detergen berkualitas, pakaian harum dan siap digunakan.</p>
            <div class="service-price">{formatCurrency(pricePerKg)}<small> / kg</small></div>
            <button class="outline-button" onclick={() => { form.ironing = false; showOrderModal = true; }}>Pilih layanan</button>
          </div>

          <div class="service-card featured-service">
            <div class="large-service-icon teal-bg"><svg viewBox="0 0 24 24" width="24" height="24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.iron}</svg></div>
            <div class="service-tag teal-tag">Disarankan</div>
            <h3>Wash &amp; Iron</h3>
            <p>Pakaian dicuci, dikeringkan, dan disetrika rapi untuk hasil maksimal.</p>
            <div class="service-price">{formatCurrency(pricePerKg + ironingPrice)}<small> / kg</small></div>
            <button class="primary-button full-button" onclick={() => { form.ironing = true; showOrderModal = true; }}>Pilih layanan</button>
          </div>

          <div class="service-card">
            <div class="large-service-icon orange-bg"><svg viewBox="0 0 24 24" width="24" height="24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.bolt}</svg></div>
            <div class="service-tag orange-tag">Cepat</div>
            <h3>Express</h3>
            <p>Prioritas pengerjaan untuk kamu yang butuh pakaian lebih cepat.</p>
            <div class="service-price">{formatCurrency(pricePerKg + expressPrice)}<small> / kg</small></div>
            <button class="outline-button" onclick={() => { form.express = true; showOrderModal = true; }}>Pilih layanan</button>
          </div>
        </div>

        <div class="info-banner">
          <div class="info-icon"><svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.truck}</svg></div>
          <div class="info-text">
            <strong>Butuh layanan penjemputan & pengantaran?</strong>
            <p>Atur tanggal dan jam pickup ({formatCurrency(pickupPrice)}) serta pengantaran ({formatCurrency(deliveryPrice)}) langsung saat memesan.</p>
          </div>
          <button onclick={() => (showOrderModal = true)}>Pesan sekarang</button>
        </div>
      </div>

    {:else if activePage === 'promo'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Hemat lebih banyak</div>
            <h1>Promo &amp; voucher</h1>
            <p>Pakai kode voucher atau pilih promo yang sedang berlaku.</p>
          </div>
        </section>

        <div class="code-card">
          <div class="info-icon"><svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.tag}</svg></div>
          <div class="code-text">
            <strong>Punya kode voucher?</strong>
            <p>Masukkan kodenya, lalu lanjut membuat pesanan.</p>
          </div>
          <div class="code-input">
            <input aria-label="Kode voucher" placeholder="Contoh: HEMAT10" bind:value={voucherInput} onkeydown={(e) => { if (e.key === 'Enter') { showOrderModal = true; applyVoucher(); } }} />
            <button class="primary-button" onclick={() => { showOrderModal = true; applyVoucher(); }}>Pakai</button>
          </div>
        </div>

        {#if vouchers.length}
          <div class="voucher-grid">
            {#each vouchers as v (v.id || v.code)}
              <div class="voucher-card">
                <div class="voucher-value"><strong>{voucherValueLabel(v)}</strong><small>OFF</small></div>
                <div class="voucher-body">
                  <h3>{v.title}</h3>
                  {#if v.description}<p>{v.description}</p>{/if}
                  <ul>
                    {#if Number(v.min_order) > 0}<li>Min. pesanan {formatCurrency(v.min_order)}</li>{/if}
                    {#if v.max_discount}<li>Maks. diskon {formatCurrency(v.max_discount)}</li>{/if}
                    {#if v.valid_until}<li>Berlaku s/d {formatDay(v.valid_until)}</li>{/if}
                  </ul>
                  <div class="voucher-foot">
                    <span class="code-chip">{v.code}</span>
                    <button class="small-btn solid" onclick={() => usePromo(v)}>Pakai</button>
                  </div>
                </div>
              </div>
            {/each}
          </div>
        {:else}
          <div class="table-card">
            <div class="empty-state">
              <div class="empty-ico"><svg viewBox="0 0 24 24" width="28" height="28" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round">{@html icons.tag}</svg></div>
              <h3>Belum ada promo aktif</h3>
              <p>Cek lagi nanti ya, promo baru akan muncul di sini.</p>
            </div>
          </div>
        {/if}
      </div>

    {:else if activePage === 'loyalty'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Program member</div>
            <h1>Poin loyalty</h1>
            <p>Kumpulkan poin tiap transaksi, tukar jadi potongan harga.</p>
          </div>
          <button class="primary-button" onclick={() => (showOrderModal = true)}>
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">{@html icons.plus}</svg>
            Pesan & kumpulkan poin
          </button>
        </section>

        <section class="loyalty-hero">
          <div class="lh-main">
            <span class="lh-label">Saldo poin kamu</span>
            <div class="lh-points">{pointsBalance}<small> poin</small></div>
            <p>Setara potongan <b>{formatCurrency(pointsBalance * pointValue)}</b></p>
          </div>
          <div class="lh-tier">
            <div class="tier-badge">{currentTier.name}</div>
            <div class="progress light"><i style="width:{tierProgress}%"></i></div>
            <small>{nextTier ? `${nextTier.min - pointsEarned} poin lagi menuju ${nextTier.name}` : 'Kamu di level tertinggi 🎉'}</small>
          </div>
        </section>

        <section class="stats-grid three">
          <div class="stat-card">
            <div class="stat-icon green"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.check}</svg></div>
            <div><span class="stat-label">Total poin diperoleh</span><h2>{pointsEarned}</h2></div>
          </div>
          <div class="stat-card">
            <div class="stat-icon orange"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.clock}</svg></div>
            <div><span class="stat-label">Poin menunggu</span><h2>{pointsPending}</h2></div>
          </div>
          <div class="stat-card">
            <div class="stat-icon purple"><svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.gift}</svg></div>
            <div><span class="stat-label">Poin terpakai</span><h2>{pointsSpent}</h2></div>
          </div>
        </section>

        <div class="two-col">
          <div class="panel">
            <h3>Cara kerja</h3>
            <ul class="how-list">
              <li>Dapatkan <b>1 poin</b> setiap pembayaran Rp{earnPerRupiah.toLocaleString('id-ID')}.</li>
              <li>Poin masuk saat status pesanan <b>Selesai</b>.</li>
              <li><b>1 poin = {formatCurrency(pointValue)}</b>. Tukar saat checkout (maks. 50% dari tagihan).</li>
              <li>Level: {tiers.map(t => `${t.name} (${t.min}+)`).join(' · ')}.</li>
            </ul>
            <h3 class="mt">Contoh penukaran</h3>
            <div class="reward-list">
              {#each rewards as r}
                <div class="reward-item" class:ok={pointsBalance >= r.points}>
                  <span>{r.points} poin</span>
                  <strong>Potongan {formatCurrency(r.value)}</strong>
                </div>
              {/each}
            </div>
          </div>

          <div class="panel">
            <h3>Riwayat poin</h3>
            {#if pointHistory.length === 0}
              <p class="muted">Belum ada riwayat poin. Buat pesanan pertamamu!</p>
            {:else}
              <div class="history-list">
                {#each pointHistory as o (o.uuid)}
                  <div class="history-item">
                    <div>
                      <strong>{o.id}</strong>
                      <small>{o.date} · {o.status}</small>
                    </div>
                    <div class="history-pts">
                      {#if o.pointsEarned > 0 && o.status !== 'Dibatalkan'}<span class="pts plus">+{o.pointsEarned}{o.status === 'Selesai' ? '' : ' (menunggu)'}</span>{/if}
                      {#if o.pointsUsed > 0 && o.status !== 'Dibatalkan'}<span class="pts minus">−{o.pointsUsed}</span>{/if}
                    </div>
                  </div>
                {/each}
              </div>
            {/if}
          </div>
        </div>
      </div>

    {:else if activePage === 'profile'}
      <div class="page-content">
        <section class="welcome-section">
          <div>
            <div class="eyebrow">Pengaturan akun</div>
            <h1>Profil saya</h1>
            <p>Informasi akun dan preferensi kamu.</p>
          </div>
        </section>

        <div class="profile-card">
          <div class="profile-cover"></div>
          <div class="profile-main">
            <div class="profile-avatar">{customer.initials}</div>
            <div class="profile-details">
              <h2>{customer.name}</h2>
              <p>{customer.email}</p>
              <span class="member-badge">Member {currentTier.name}</span>
            </div>
          </div>

          <div class="profile-fields">
            <div class="profile-field"><span class="pf-label">Nama lengkap</span><div>{customer.name}</div></div>
            <div class="profile-field"><span class="pf-label">Email</span><div class="wrap-any">{customer.email}</div></div>
            <div class="profile-field"><span class="pf-label">Status akun</span><div><span class="status-badge completed"><span class="status-dot"></span>Aktif</span></div></div>
            <div class="profile-field"><span class="pf-label">Member sejak</span><div>{currentUser?.created_at ? formatDate(currentUser.created_at) : '-'}</div></div>
            <div class="profile-field"><span class="pf-label">Saldo poin</span><div>{pointsBalance} poin</div></div>
            <div class="profile-field"><span class="pf-label">Ulasan diberikan</span><div>{Object.keys(feedbacks).length}</div></div>
          </div>
        </div>
      </div>
    {/if}

    <footer class="footer">
      <span>© {new Date().getFullYear()} LaundryKu. All rights reserved.</span>
      <span>Dibuat dengan teliti untuk hidup yang lebih bersih.</span>
    </footer>
  </main>

  <!-- BOTTOM NAV (mobile & tablet) -->
  <nav class="bottom-nav" aria-label="Menu utama">
    {#each navItems as item}
      <button class:active={activePage === item.id} aria-current={activePage === item.id ? 'page' : undefined} onclick={() => navigate(item.id)}>
        <svg viewBox="0 0 24 24" width="21" height="21" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons[item.icon]}</svg>
        <span>{item.short}</span>
      </button>
    {/each}
  </nav>

  <button class="fab" aria-label="Pesan laundry" onclick={() => (showOrderModal = true)}>
    <svg viewBox="0 0 24 24" width="24" height="24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">{@html icons.plus}</svg>
  </button>
</div>

<!-- TOAST -->
<div class="toast-zone" aria-live="polite">
  {#if toast}
    <div class="toast {toast.type}" role={toast.type === 'error' ? 'alert' : 'status'}>
      <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html toast.type === 'error' ? icons.info : icons.check}</svg>
      <span>{toast.text}</span>
    </div>
  {/if}
</div>

<!-- ORDER MODAL -->
{#if showOrderModal}
  <div class="modal-overlay" role="presentation" onclick={(e) => { if (e.target === e.currentTarget) showOrderModal = false; }}>
    <div class="modal" role="dialog" aria-modal="true" aria-labelledby="order-modal-title">
      <div class="modal-header">
        <div>
          <div class="eyebrow">Pesanan baru</div>
          <h2 id="order-modal-title">Pesan laundry</h2>
          <p>Lengkapi detail laundry kamu.</p>
        </div>
        <button class="close-button" aria-label="Tutup" onclick={() => (showOrderModal = false)}>
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round">{@html icons.close}</svg>
        </button>
      </div>

      <div class="modal-body">
        <div class="form-group">
          <label for="weight">Berat pakaian</label>
          <div class="weight-input">
            <button type="button" aria-label="Kurangi" onclick={() => (form.weight = Math.max(1, Number(form.weight) - 1))}>−</button>
            <input id="weight" type="number" min="1" inputmode="numeric" bind:value={form.weight} />
            <button type="button" aria-label="Tambah" onclick={() => (form.weight = Number(form.weight) + 1)}>+</button>
            <span class="kg-label">KG</span>
          </div>
          <small class="field-hint">Estimasi berat pakaian kamu.</small>
        </div>

        <div class="form-group">
          <span class="group-label">Layanan setrika</span>
          <div class="option-grid">
            <button type="button" class="option-card" class:selected={!form.ironing} aria-pressed={!form.ironing} onclick={() => (form.ironing = false)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.drop}</svg>
              <strong>Cuci saja</strong><small>Rp5.000/kg</small>
            </button>
            <button type="button" class="option-card" class:selected={form.ironing} aria-pressed={form.ironing} onclick={() => (form.ironing = true)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.iron}</svg>
              <strong>Cuci + setrika</strong><small>Rp7.000/kg</small>
            </button>
          </div>
        </div>

        <div class="form-group">
          <span class="group-label">Kecepatan pengerjaan</span>
          <div class="option-grid">
            <button type="button" class="option-card" class:selected={!form.express} aria-pressed={!form.express} onclick={() => (form.express = false)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.clock}</svg>
              <strong>Regular</strong><small>2–3 hari</small>
            </button>
            <button type="button" class="option-card" class:selected={form.express} aria-pressed={form.express} onclick={() => (form.express = true)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.bolt}</svg>
              <strong>Express</strong><small>1 hari · +Rp3.000/kg</small>
            </button>
          </div>
        </div>

        <!-- PENJEMPUTAN -->
        <div class="form-group">
          <span class="group-label">Penjemputan pakaian</span>
          <div class="option-grid">
            <button type="button" class="option-card" class:selected={!form.pickup} aria-pressed={!form.pickup} onclick={() => (form.pickup = false)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.home2}</svg>
              <strong>Antar sendiri</strong><small>Drop-off ke laundry</small>
            </button>
            <button type="button" class="option-card" class:selected={form.pickup} aria-pressed={form.pickup} onclick={() => (form.pickup = true)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.truck}</svg>
              <strong>Pickup</strong><small>+Rp5.000</small>
            </button>
          </div>
        </div>

        {#if form.pickup}
          <div class="pickup-details">
            <div class="form-group">
              <label for="address">Alamat penjemputan <span class="required">*</span></label>
              <textarea id="address" rows="3" placeholder="Nama jalan, nomor rumah, kelurahan..." bind:value={form.address}></textarea>
            </div>

            <div class="form-group">
              <div class="two-fields">
                <div>
                  <label class="field-label" for="pickup-date">Tanggal jemput <span class="required">*</span></label>
                  <input id="pickup-date" class="field-input" type="date" min={today} bind:value={form.pickupDate} />
                </div>
                <div>
                  <label class="field-label" for="pickup-time">Jam jemput <span class="required">*</span></label>
                  <select id="pickup-time" class="field-input" bind:value={form.pickupTime}>
                    <option value="">Pilih jam</option>
                    {#each timeSlots as slot}<option value={slot}>{slot}</option>{/each}
                  </select>
                </div>
              </div>
            </div>

            <div class="form-group">
              <span class="group-label">Foto pakaian</span>
              <label class="upload-box">
                {#if form.photo}
                  <img src={form.photo} alt="Preview pakaian" />
                  <div class="upload-overlay">Ganti foto</div>
                {:else}
                  <div class="upload-content">
                    <div class="upload-icon"><svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.upload}</svg></div>
                    <strong>Upload foto pakaian</strong>
                    <small>PNG, JPG hingga 5 MB</small>
                  </div>
                {/if}
                <input type="file" accept="image/*" onchange={handlePhoto} hidden />
              </label>
              {#if form.photoName}<small class="uploaded-name">✓ {form.photoName}</small>{/if}
            </div>

            <div class="form-group">
              <label for="notes">Catatan penjemputan</label>
              <textarea id="notes" rows="2" placeholder="Contoh: pakaian ada di depan rumah..." bind:value={form.notes}></textarea>
            </div>
          </div>
        {/if}

        <!-- PENGANTARAN -->
        <div class="form-group">
          <span class="group-label">Pengambilan setelah selesai</span>
          <div class="option-grid">
            <button type="button" class="option-card" class:selected={!form.delivery} aria-pressed={!form.delivery} onclick={() => (form.delivery = false)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.home2}</svg>
              <strong>Ambil sendiri</strong><small>Ambil di laundry</small>
            </button>
            <button type="button" class="option-card" class:selected={form.delivery} aria-pressed={form.delivery} onclick={() => (form.delivery = true)}>
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons.truck}</svg>
              <strong>Diantar</strong><small>+Rp5.000</small>
            </button>
          </div>
        </div>

        {#if form.delivery}
          <div class="pickup-details">
            {#if form.pickup}
              <label class="check-row">
                <input type="checkbox" bind:checked={form.deliverySame} />
                <span>Alamat pengantaran sama dengan alamat penjemputan</span>
              </label>
            {/if}

            {#if !(form.pickup && form.deliverySame)}
              <div class="form-group">
                <label for="delivery-address">Alamat pengantaran <span class="required">*</span></label>
                <textarea id="delivery-address" rows="3" placeholder="Nama jalan, nomor rumah, kelurahan..." bind:value={form.deliveryAddress}></textarea>
              </div>
            {/if}

            <div class="form-group">
              <div class="two-fields">
                <div>
                  <label class="field-label" for="delivery-date">Tanggal antar <span class="required">*</span></label>
                  <input id="delivery-date" class="field-input" type="date" min={minDelivery} bind:value={form.deliveryDate} />
                </div>
                <div>
                  <label class="field-label" for="delivery-time">Jam antar <span class="required">*</span></label>
                  <select id="delivery-time" class="field-input" bind:value={form.deliveryTime}>
                    <option value="">Pilih jam</option>
                    {#each timeSlots as slot}<option value={slot}>{slot}</option>{/each}
                  </select>
                </div>
              </div>
              <small class="field-hint">Paling cepat {formatDay(minDelivery)} ({form.express ? 'Express 1 hari' : 'Regular 3 hari'}).</small>
            </div>
          </div>
        {/if}

        <!-- VOUCHER -->
        <div class="form-group">
          <label for="voucher">Kode voucher</label>
          <div class="voucher-input">
            <input id="voucher" class="field-input" placeholder="Masukkan kode (mis. HEMAT10)" bind:value={voucherInput} onkeydown={(e) => { if (e.key === 'Enter') { e.preventDefault(); applyVoucher(); } }} />
            {#if appliedVoucher}
              <button type="button" class="cancel-button" onclick={removeVoucher}>Hapus</button>
            {:else}
              <button type="button" class="outline-button apply" disabled={checkingVoucher} onclick={() => applyVoucher()}>{checkingVoucher ? '...' : 'Pakai'}</button>
            {/if}
          </div>
          {#if voucherMsg}<small class="voucher-msg" class:ok={voucherOk}>{voucherMsg}</small>{/if}
          {#if !appliedVoucher && vouchers.length}
            <div class="chip-row">
              {#each vouchers.slice(0, 3) as v}
                <button type="button" class="code-chip clickable" onclick={() => { voucherInput = v.code; applyVoucher(v.code); }}>{v.code} · {voucherValueLabel(v)}</button>
              {/each}
            </div>
          {/if}
        </div>

        <!-- POIN -->
        {#if pointsBalance > 0}
          <label class="check-row points-row">
            <input type="checkbox" bind:checked={usePoints} disabled={maxPoints === 0} />
            <span>
              Tukar poin ({pointsBalance} tersedia)
              <small>{maxPoints > 0 ? `Pakai ${maxPoints} poin, hemat ${formatCurrency(maxPoints * pointValue)}` : 'Belum bisa dipakai untuk total ini'}</small>
            </span>
          </label>
        {/if}

        <!-- PEMBAYARAN -->
        <div class="form-group">
          <span class="group-label">Metode pembayaran</span>
          <div class="option-grid three">
            {#each paymentMethods as m}
              <button type="button" class="option-card" class:selected={form.paymentMethod === m.id} aria-pressed={form.paymentMethod === m.id} onclick={() => (form.paymentMethod = m.id)}>
                <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons[m.icon]}</svg>
                <strong>{m.label}</strong><small>{m.hint}</small>
              </button>
            {/each}
          </div>
        </div>

        <div class="price-summary">
          <div class="summary-title"><strong>Ringkasan pembayaran</strong><span>Estimasi</span></div>
          <div class="summary-row"><span>Cuci ({form.weight || 0} kg)</span><span>{formatCurrency(washAmount)}</span></div>
          {#if form.ironing}<div class="summary-row"><span>Setrika</span><span>{formatCurrency(ironAmount)}</span></div>{/if}
          {#if form.express}<div class="summary-row"><span>Express</span><span>{formatCurrency(expressAmount)}</span></div>{/if}
          {#if form.pickup}<div class="summary-row"><span>Pickup</span><span>{formatCurrency(pickupPrice)}</span></div>{/if}
          {#if form.delivery}<div class="summary-row"><span>Pengantaran</span><span>{formatCurrency(deliveryPrice)}</span></div>{/if}
          {#if voucherDiscount > 0}<div class="summary-row discount"><span>Voucher {appliedVoucher.code}</span><span>−{formatCurrency(voucherDiscount)}</span></div>{/if}
          {#if pointsDiscount > 0}<div class="summary-row discount"><span>Tukar {pointsUsed} poin</span><span>−{formatCurrency(pointsDiscount)}</span></div>{/if}
          <div class="summary-total"><span>Total estimasi</span><strong>{formatCurrency(total)}</strong></div>
          <div class="earn-hint">Kamu akan dapat <b>+{pointsEarnEstimate} poin</b> setelah pesanan selesai.</div>
        </div>
      </div>

      <div class="modal-footer">
        <button class="cancel-button" onclick={() => (showOrderModal = false)}>Batal</button>
        <button class="primary-button submit-button" disabled={savingOrder} onclick={submitOrder}>
          {savingOrder ? 'Menyimpan...' : 'Konfirmasi pesanan'}
        </button>
      </div>
    </div>
  </div>
{/if}

<!-- DETAIL / PEMBAYARAN / REVIEW -->
{#if showDetail || showPay || showReview}
  <div class="modal-overlay" role="presentation" onclick={(e) => { if (e.target === e.currentTarget) { showDetail = null; showPay = null; showReview = null; } }}>

    {#if showDetail}
      <div class="modal receipt-modal" role="dialog" aria-modal="true" aria-labelledby="detail-title">
        <div class="modal-header no-print">
          <div>
            <div class="eyebrow">Detail pesanan</div>
            <h2 id="detail-title">{showDetail.id}</h2>
            <p>{showDetail.date} · {showDetail.service} · {showDetail.weight} kg</p>
          </div>
          <button class="close-button" aria-label="Tutup" onclick={() => (showDetail = null)}>
            <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round">{@html icons.close}</svg>
          </button>
        </div>

        <div class="modal-body">
          <div class="print-only receipt-brand"><strong>LaundryKu</strong><span>Bukti transaksi · {showDetail.id} · {showDetail.date}</span></div>

          <div class="detail-badges">
            <span class="status-badge {showDetail.statusColor}"><span class="status-dot"></span>Pesanan: {showDetail.status}</span>
            <span class="status-badge {showDetail.paymentColor}"><span class="status-dot"></span>Bayar: {showDetail.paymentStatus}</span>
          </div>

          <div class="detail-block">
            <h4>Jadwal</h4>
            <div class="detail-grid">
              <div>
                <span class="pf-label">Penjemputan</span>
                {#if showDetail.pickup}
                  <b>{formatDay(showDetail.pickupDate)}, {showDetail.pickupTime}</b>
                  <p>{showDetail.address}</p>
                {:else}<b>Diantar sendiri ke laundry</b>{/if}
              </div>
              <div>
                <span class="pf-label">Pengantaran</span>
                {#if showDetail.delivery}
                  <b>{formatDay(showDetail.deliveryDate)}, {showDetail.deliveryTime}</b>
                  <p>{showDetail.deliveryAddress}</p>
                {:else}<b>Diambil sendiri di laundry</b>{/if}
              </div>
            </div>
            {#if showDetail.notes}<p class="detail-note">Catatan: {showDetail.notes}</p>{/if}
          </div>

          <div class="detail-block">
            <h4>Bukti transaksi</h4>
            <div class="receipt">
              {#each receiptLines(showDetail) as line}
                <div class="summary-row" class:discount={line.value < 0}>
                  <span>{line.label}</span><span>{line.value < 0 ? '−' : ''}{formatCurrency(Math.abs(line.value))}</span>
                </div>
              {/each}
              <div class="summary-total"><span>Total</span><strong>{formatCurrency(showDetail.total)}</strong></div>
              <div class="receipt-meta">
                <span>Metode: {methodLabel[showDetail.paymentMethod] || '-'}</span>
                <span>Status: {showDetail.paymentStatus}</span>
                {#if showDetail.proofUrl}<span>Bukti pembayaran: terunggah ✓</span>{/if}
                {#if showDetail.pointsEarned}<span>Poin: +{showDetail.pointsEarned}{showDetail.pointsUsed ? ` / −${showDetail.pointsUsed}` : ''}</span>{/if}
              </div>
            </div>
          </div>

          {#if feedbacks[showDetail.uuid]}
            <div class="detail-block no-print">
              <h4>Ulasanmu</h4>
              <div class="stars-static" aria-label="Rating {feedbacks[showDetail.uuid].rating} dari 5">
                {#each [1, 2, 3, 4, 5] as n}
                  <svg viewBox="0 0 24 24" width="20" height="20" fill={n <= feedbacks[showDetail.uuid].rating ? 'currentColor' : 'none'} stroke="currentColor" stroke-width="1.8" stroke-linejoin="round">{@html icons.star}</svg>
                {/each}
              </div>
              {#if feedbacks[showDetail.uuid].comment}<p class="detail-note">“{feedbacks[showDetail.uuid].comment}”</p>{/if}
            </div>
          {/if}
        </div>

        <div class="modal-footer no-print">
          <button class="cancel-button" onclick={() => window.print()}>
            <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="vertical-align:-3px;margin-right:6px">{@html icons.print}</svg>Cetak bukti
          </button>
          {#if showDetail.status === 'Selesai'}
            <button class="cancel-button" onclick={() => openReview(showDetail)}>{feedbacks[showDetail.uuid] ? 'Ubah ulasan' : 'Beri ulasan'}</button>
          {/if}
          {#if showDetail.paymentStatus === 'Belum Dibayar' && showDetail.status !== 'Dibatalkan'}
            <button class="primary-button" onclick={() => openPay(showDetail)}>Bayar sekarang</button>
          {/if}
        </div>
      </div>
    {/if}

    {#if showPay}
      <div class="modal" role="dialog" aria-modal="true" aria-labelledby="pay-title">
        <div class="modal-header">
          <div>
            <div class="eyebrow">Pembayaran</div>
            <h2 id="pay-title">Bayar pesanan {showPay.id}</h2>
            <p>Total tagihan <b>{formatCurrency(showPay.total)}</b></p>
          </div>
          <button class="close-button" aria-label="Tutup" onclick={() => (showPay = null)}>
            <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round">{@html icons.close}</svg>
          </button>
        </div>

        <div class="modal-body">
          <div class="form-group">
            <span class="group-label">Pilih metode</span>
            <div class="option-grid three">
              {#each paymentMethods as m}
                <button type="button" class="option-card" class:selected={payMethod === m.id} aria-pressed={payMethod === m.id} onclick={() => (payMethod = m.id)}>
                  <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round">{@html icons[m.icon]}</svg>
                  <strong>{m.label}</strong><small>{m.hint}</small>
                </button>
              {/each}
            </div>
          </div>

          {#if payMethod === 'qris'}
            <div class="pay-box">
              {#if qrisImage}
                <img class="qris-img" src={qrisImage} alt="QRIS LaundryKu" />
              {:else}
                <div class="qris-placeholder">
                  <svg viewBox="0 0 24 24" width="48" height="48" fill="none" stroke="currentColor" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round">{@html icons.qr}</svg>
                  <small>Pasang gambar QRIS toko di variabel <code>qrisImage</code></small>
                </div>
              {/if}
              <p>Scan QRIS, bayar tepat <b>{formatCurrency(showPay.total)}</b>, lalu unggah bukti di bawah.</p>
            </div>
          {:else if payMethod === 'transfer'}
            <div class="pay-box">
              {#each bankAccounts as acc}
                <div class="bank-row">
                  <div>
                    <strong>{acc.bank}</strong>
                    <span>{acc.number}</span>
                    <small>a.n. {acc.name}</small>
                  </div>
                  <button type="button" class="small-btn" onclick={() => copyText(acc.number)}>
                    <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="vertical-align:-2px;margin-right:4px">{@html icons.copy}</svg>Salin
                  </button>
                </div>
              {/each}
              <p>Transfer tepat <b>{formatCurrency(showPay.total)}</b>, lalu unggah bukti di bawah.</p>
            </div>
          {:else}
            <div class="pay-box">
              <p>Kamu akan membayar <b>{formatCurrency(showPay.total)}</b> langsung ke kasir saat laundry selesai atau diantar.</p>
            </div>
          {/if}

          {#if payMethod !== 'cod'}
            <div class="form-group">
              <span class="group-label">Bukti pembayaran <span class="required">*</span></span>
              <label class="upload-box">
                {#if proofPreview}
                  <img src={proofPreview} alt="Preview bukti pembayaran" />
                  <div class="upload-overlay">Ganti bukti</div>
                {:else}
                  <div class="upload-content">
                    <div class="upload-icon"><svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{@html icons.upload}</svg></div>
                    <strong>{showPay.proofUrl ? 'Bukti sebelumnya sudah terunggah — ganti?' : 'Upload bukti transfer / struk'}</strong>
                    <small>PNG, JPG hingga 5 MB</small>
                  </div>
                {/if}
                <input type="file" accept="image/*" onchange={handleProof} hidden />
              </label>
            </div>
          {/if}
        </div>

        <div class="modal-footer">
          <button class="cancel-button" onclick={() => (showPay = null)}>Nanti saja</button>
          <button class="primary-button submit-button" disabled={paying} onclick={confirmPay}>
            {paying ? 'Memproses...' : payMethod === 'cod' ? 'Simpan metode' : 'Kirim bukti pembayaran'}
          </button>
        </div>
      </div>
    {/if}

    {#if showReview}
      <div class="modal" role="dialog" aria-modal="true" aria-labelledby="review-title">
        <div class="modal-header">
          <div>
            <div class="eyebrow">Feedback</div>
            <h2 id="review-title">Bagaimana laundry kamu?</h2>
            <p>Pesanan {showReview.id} · {showReview.service}</p>
          </div>
          <button class="close-button" aria-label="Tutup" onclick={() => (showReview = null)}>
            <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round">{@html icons.close}</svg>
          </button>
        </div>

        <div class="modal-body">
          {#each ratingFields as f}
            <div class="rate-row" class:big={f.big}>
              <span class="group-label">{f.label}</span>
              <div class="stars" role="radiogroup" aria-label={f.label}>
                {#each [1, 2, 3, 4, 5] as n}
                  <button type="button" role="radio" aria-checked={review[f.key] === n} aria-label="{n} bintang" class:on={n <= review[f.key]} onclick={() => (review[f.key] = n)}>
                    <svg viewBox="0 0 24 24" width={f.big ? 34 : 24} height={f.big ? 34 : 24} fill={n <= review[f.key] ? 'currentColor' : 'none'} stroke="currentColor" stroke-width="1.7" stroke-linejoin="round">{@html icons.star}</svg>
                  </button>
                {/each}
              </div>
            </div>
          {/each}

          <div class="form-group">
            <label for="review-comment">Komentar (opsional)</label>
            <textarea id="review-comment" class="field-input" rows="3" placeholder="Ceritakan pengalamanmu: kebersihan, aroma, kerapian, ketepatan waktu..." bind:value={review.comment}></textarea>
          </div>
        </div>

        <div class="modal-footer">
          <button class="cancel-button" onclick={() => (showReview = null)}>Batal</button>
          <button class="primary-button submit-button" disabled={savingReview} onclick={submitReview}>
            {savingReview ? 'Menyimpan...' : 'Kirim ulasan'}
          </button>
        </div>
      </div>
    {/if}
  </div>
{/if}

<style>
  :global(*) { box-sizing: border-box; }

  :global(body) {
    margin: 0;
    background: #f1f6f9;
    color: #0c2b45;
    font-family: 'Plus Jakarta Sans', Inter, 'Segoe UI', Arial, sans-serif;
    font-size: 14px;
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

  :global(button), :global(input), :global(textarea), :global(select) { font: inherit; }
  :global(button) { cursor: pointer; }
  button:focus-visible, input:focus-visible, textarea:focus-visible, select:focus-visible, label:focus-within {
    outline: 3px solid rgba(14, 124, 134, 0.4);
    outline-offset: 2px;
  }
  h1, h2, h3, p { margin-top: 0; }

  .app-shell { min-height: 100vh; min-height: 100dvh; display: flex; }
  .print-only { display: none; }

  /* ================= SIDEBAR ================= */
  .sidebar {
    position: sticky; top: 0; align-self: flex-start;
    width: 264px; height: 100vh; height: 100dvh; flex-shrink: 0;
    display: flex; flex-direction: column; padding: 24px 16px 18px;
    color: #e3f1f6;
    background:
      radial-gradient(360px 240px at 100% 0, rgba(43, 196, 192, 0.22), transparent 70%),
      linear-gradient(170deg, #0c2b45, #082238);
    overflow-y: auto;
  }
  .brand { display: flex; align-items: center; gap: 12px; padding: 4px 10px 22px; }
  .brand-icon {
    width: 40px; height: 40px; border-radius: 12px; display: grid; place-items: center; color: #fff;
    background: linear-gradient(145deg, var(--aqua), var(--teal));
    box-shadow: 0 8px 20px rgba(43, 196, 192, 0.3);
  }
  .brand-icon.sm { width: 34px; height: 34px; border-radius: 10px; }
  .brand-text strong { display: block; font-size: 19px; font-weight: 800; letter-spacing: -0.3px; color: #fff; }
  .brand-text small { color: #86a8b8; font-size: 12px; }

  .nav-label { color: #6f94a6; font-size: 12px; font-weight: 600; margin: 4px 12px 10px; }
  .navigation { display: flex; flex-direction: column; gap: 4px; }
  .navigation button {
    display: flex; align-items: center; gap: 12px; width: 100%;
    border: 0; background: transparent; color: #b9d2de;
    padding: 11px 12px; border-radius: 12px; text-align: left; font-size: 14px; font-weight: 600;
    transition: background 0.18s, color 0.18s;
  }
  .navigation button:hover { background: rgba(255, 255, 255, 0.07); color: #fff; }
  .navigation button.active { background: rgba(43, 196, 192, 0.16); color: #fff; box-shadow: inset 3px 0 0 var(--aqua); }
  .navigation button.active svg { color: var(--aqua); }
  .nav-count { margin-left: auto; background: rgba(255, 255, 255, 0.12); color: #e3f1f6; border-radius: 20px; padding: 3px 9px; font-size: 11px; }

  .sidebar-promo {
    margin-top: auto; margin-bottom: 16px; padding: 18px; border-radius: 16px;
    background: rgba(255, 255, 255, 0.07); border: 1px solid rgba(255, 255, 255, 0.1);
  }
  .sidebar-promo strong { font-size: 15px; line-height: 1.45; color: #fff; }
  .sidebar-promo p { color: #9fbfce; font-size: 12.5px; line-height: 1.65; margin: 8px 0 14px; }
  .sidebar-promo button {
    display: inline-flex; align-items: center; gap: 8px; border: 0; padding: 0;
    background: transparent; color: var(--aqua); font-weight: 700; font-size: 13px;
  }

  .sidebar-bottom { border-top: 1px solid rgba(255, 255, 255, 0.1); padding-top: 14px; }
  .sidebar-user { display: flex; align-items: center; gap: 11px; padding: 4px 4px; }
  .avatar {
    flex: none; width: 38px; height: 38px; border-radius: 12px; display: grid; place-items: center;
    background: rgba(43, 196, 192, 0.2); color: var(--aqua); font-size: 13px; font-weight: 800;
  }
  .user-info { flex: 1; min-width: 0; }
  .user-info strong { display: block; font-size: 13px; color: #fff; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  .user-info small { color: #86a8b8; font-size: 12px; display: block; margin-top: 2px; }

  .icon-btn {
    flex: none; width: 38px; height: 38px; display: grid; place-items: center;
    border: 0; border-radius: 11px; background: transparent; color: #b9d2de; position: relative;
    transition: background 0.18s;
  }
  .icon-btn:hover { background: rgba(255, 255, 255, 0.1); color: #fff; }
  .icon-btn.bordered { border: 1.5px solid var(--line); background: #fff; color: var(--muted); }
  .icon-btn.bordered:hover { background: #f6fafc; color: var(--ink); }

  /* ================= MAIN ================= */
  .main-content { flex: 1; min-width: 0; display: flex; flex-direction: column; }

  .topbar {
    position: sticky; top: 0; z-index: 8; height: 72px; padding: 0 clamp(20px, 3vw, 40px);
    display: flex; align-items: center; justify-content: space-between;
    background: rgba(255, 255, 255, 0.9); backdrop-filter: blur(10px);
    border-bottom: 1px solid var(--line);
  }
  .mobile-brand { display: none; align-items: center; gap: 10px; font-size: 17px; font-weight: 800; letter-spacing: -0.3px; }
  .breadcrumb { font-size: 13px; color: var(--soft); }
  .breadcrumb strong { color: var(--ink); font-weight: 700; }
  .crumb-separator { margin: 0 10px; color: #cbd8e1; }

  .topbar-actions { display: flex; align-items: center; gap: 12px; position: relative; }
  .mobile-only { display: none; }
  .date-chip {
    display: flex; align-items: center; gap: 8px; color: var(--muted); font-size: 13px; font-weight: 500;
    background: #f6fafc; border: 1px solid var(--line); border-radius: 11px; padding: 9px 13px;
  }
  .notification-dot {
    width: 9px; height: 9px; background: #ff6b5e; border: 2px solid #fff; border-radius: 50%;
    position: absolute; top: 8px; right: 9px;
  }
  .notification-popup {
    position: absolute; top: 50px; right: 0; z-index: 10; width: 280px; padding: 16px;
    background: #fff; border: 1px solid var(--line); border-radius: 14px;
    box-shadow: 0 18px 40px rgba(12, 43, 69, 0.15); animation: pop 0.2s ease;
  }
  .notification-popup p { color: var(--muted); font-size: 13px; line-height: 1.6; margin: 8px 0 0; }
  .link-btn { border: 0; background: transparent; padding: 0; color: var(--teal); font-weight: 700; text-decoration: underline; font-size: 13px; }
  @keyframes pop { from { opacity: 0; transform: translateY(-4px); } to { opacity: 1; transform: none; } }

  .page-content { padding: 30px clamp(20px, 3vw, 40px); flex: 1; width: 100%; max-width: 1400px; }

  .welcome-section { display: flex; justify-content: space-between; align-items: center; gap: 20px; margin-bottom: 24px; }
  .eyebrow { color: var(--teal); font-size: 13px; font-weight: 700; margin-bottom: 8px; }
  h1 { font-size: 28px; letter-spacing: -0.9px; margin-bottom: 6px; font-weight: 800; line-height: 1.15; }
  .welcome-section p { color: var(--muted); font-size: 14.5px; margin-bottom: 0; }

  .primary-button {
    display: inline-flex; align-items: center; justify-content: center; gap: 8px;
    height: 46px; padding: 0 20px; border: 0; border-radius: 13px;
    background: linear-gradient(135deg, var(--teal), #0b5f78); color: #fff;
    font-size: 14px; font-weight: 700; white-space: nowrap;
    box-shadow: 0 8px 18px rgba(14, 124, 134, 0.25);
    transition: transform 0.18s, filter 0.18s, box-shadow 0.18s;
  }
  .primary-button:hover:not(:disabled) { transform: translateY(-1px); filter: brightness(1.06); box-shadow: 0 12px 24px rgba(14, 124, 134, 0.3); }
  .primary-button:disabled { opacity: 0.65; cursor: not-allowed; }

  .small-btn {
    height: 32px; padding: 0 12px; border: 1.5px solid #cfe3e6; border-radius: 10px; background: #fff;
    color: var(--teal-dark); font-size: 12.5px; font-weight: 700; white-space: nowrap; transition: all 0.18s;
  }
  .small-btn:hover { background: var(--foam); }
  .small-btn.solid { background: var(--teal); border-color: var(--teal); color: #fff; }
  .small-btn.solid:hover { background: var(--teal-dark); }

  .message.error { color: #b42318; background: #fef3f2; border: 1px solid #fcd0cc; border-radius: 12px; padding: 14px 16px; margin-bottom: 16px; }

  .state-card { text-align: center; padding: 70px 20px; }
  .state-card h3 { margin: 18px 0 6px; }
  .state-card p { color: var(--muted); margin: 0; }
  .loader { width: 42px; height: 42px; margin: 0 auto; border-radius: 50%; border: 4px solid #cfe9ea; border-top-color: var(--teal); animation: spin 0.8s linear infinite; }
  @keyframes spin { to { transform: rotate(360deg); } }

  /* ---------- stats ---------- */
  .stats-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 16px; margin-bottom: 22px; }
  .stats-grid.three { grid-template-columns: repeat(3, minmax(0, 1fr)); }
  .stat-card { display: flex; align-items: center; gap: 14px; padding: 18px 20px; background: #fff; border: 1px solid var(--line); border-radius: 16px; }
  .stat-icon { flex: none; width: 46px; height: 46px; border-radius: 13px; display: grid; place-items: center; }
  .blue { background: #e8f1fd; color: #2563c9; }
  .orange { background: #fff2e2; color: #c2620a; }
  .green { background: #e3f7ee; color: #0b8a55; }
  .purple { background: #efeafd; color: #6a45d1; }
  .stat-label { display: block; color: var(--muted); font-size: 13px; font-weight: 500; margin-bottom: 4px; }
  .stat-card h2 { font-size: 26px; font-weight: 800; letter-spacing: -0.5px; margin: 0; }
  .stat-card h2 small { font-size: 14px; color: var(--soft); font-weight: 600; }

  /* ---------- action row (dashboard) ---------- */
  .action-row { display: grid; grid-template-columns: 1.1fr 1fr; gap: 16px; margin-bottom: 22px; }
  .loyalty-mini {
    display: flex; align-items: center; gap: 14px; text-align: left; padding: 18px 20px; background: #fff;
    border: 1px solid var(--line); border-radius: 16px; color: inherit; transition: box-shadow 0.18s, transform 0.18s;
  }
  .loyalty-mini:hover { box-shadow: 0 10px 24px rgba(12, 43, 69, 0.08); transform: translateY(-1px); }
  .lm-text { flex: 1; min-width: 0; }
  .lm-text strong { display: block; font-size: 22px; font-weight: 800; letter-spacing: -0.4px; margin-bottom: 8px; }
  .lm-text strong small { font-size: 12.5px; color: var(--soft); font-weight: 600; letter-spacing: 0; }
  .lm-hint { display: block; color: var(--muted); font-size: 12px; margin-top: 7px; }
  .progress { height: 8px; border-radius: 99px; background: #e6eef3; overflow: hidden; }
  .progress i { display: block; height: 100%; border-radius: 99px; background: linear-gradient(90deg, var(--aqua), var(--teal)); transition: width 0.4s ease; }
  .progress.light { background: rgba(255, 255, 255, 0.2); }
  .progress.light i { background: linear-gradient(90deg, #fff, #9fe3e0); }

  .todo-card { padding: 18px 20px; background: #fff; border: 1px solid var(--line); border-radius: 16px; }
  .todo-card > strong { display: block; margin-bottom: 8px; font-size: 14.5px; }
  .todo-card p { color: var(--muted); margin: 6px 0 0; font-size: 13.5px; }
  .todo-item { display: flex; align-items: center; justify-content: space-between; gap: 10px; padding: 9px 0; border-bottom: 1px dashed #e5edf2; font-size: 13.5px; color: #3d566d; }
  .todo-item:last-child { border-bottom: 0; }

  /* ---------- hero ---------- */
  .hero-banner {
    position: relative; overflow: hidden; display: flex; min-height: 230px; margin-bottom: 30px;
    border-radius: 20px; color: #fff; background: #082238;
  }
  .hero-bg {
    position: absolute; inset: 0;
    background:
      radial-gradient(480px 320px at 90% 10%, rgba(43, 196, 192, 0.4), transparent 65%),
      radial-gradient(420px 380px at 10% 120%, rgba(14, 124, 134, 0.6), transparent 65%),
      linear-gradient(120deg, #0c2b45, #0a3a54);
  }
  .hero-content { position: relative; z-index: 2; width: 62%; padding: 32px 36px; }
  .hero-tag { font-size: 13px; font-weight: 600; color: #9fe3e0; margin-bottom: 14px; }
  .hero-content h2 { font-size: 27px; line-height: 1.3; letter-spacing: -0.7px; font-weight: 800; margin-bottom: 12px; }
  .hero-content p { font-size: 14px; line-height: 1.75; color: #c3dbe6; max-width: 380px; margin-bottom: 20px; }
  .hero-button {
    display: inline-flex; align-items: center; gap: 10px; border: 0; border-radius: 12px;
    padding: 12px 18px; background: #fff; color: var(--teal-dark); font-size: 14px; font-weight: 800;
    transition: transform 0.18s;
  }
  .hero-button:hover { transform: translateY(-1px); }

  .hero-decoration { position: absolute; right: 0; top: 0; width: 38%; height: 100%; }
  .drum {
    position: absolute; width: 170px; height: 170px; right: 60px; top: 30px; border-radius: 50%; overflow: hidden;
    background: #0b4a63; border: 10px solid #cfe3ec; box-shadow: 0 24px 50px rgba(0, 0, 0, 0.35);
  }
  .water { position: absolute; left: -50%; top: 40%; width: 200%; height: 200%; }
  .water i { position: absolute; left: 0; width: 100%; height: 100%; border-radius: 42%; background: rgba(43, 196, 192, 0.75); animation: spin 6s linear infinite; }
  .water i:nth-child(2) { background: rgba(255, 255, 255, 0.22); border-radius: 38%; animation-duration: 9s; animation-direction: reverse; }
  .bub { position: absolute; bottom: -10px; border-radius: 50%; background: rgba(255, 255, 255, 0.75); animation: rise 4.5s ease-in infinite; }
  .b1 { width: 12px; height: 12px; left: 24%; }
  .b2 { width: 8px; height: 8px; left: 55%; animation-delay: 1.5s; }
  .b3 { width: 14px; height: 14px; left: 72%; animation-delay: 2.8s; }
  @keyframes rise { 0% { transform: translateY(0) scale(0.6); opacity: 0; } 20% { opacity: 1; } 100% { transform: translateY(-110px) scale(1); opacity: 0; } }
  .floating-card {
    position: absolute; left: 0; bottom: 28px; display: flex; align-items: center; gap: 10px;
    padding: 11px 15px 11px 11px; border-radius: 14px; background: #fff; color: var(--ink);
    box-shadow: 0 18px 36px rgba(0, 0, 0, 0.28);
  }
  .floating-check { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; background: var(--foam); color: var(--teal); }
  .floating-card strong { display: block; font-size: 13px; }
  .floating-card small { display: block; color: var(--muted); font-size: 11px; margin-top: 2px; }

  /* ---------- orders table ---------- */
  .orders-section { margin-bottom: 20px; }
  .section-heading { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; gap: 12px; }
  .section-heading h2 { font-size: 18px; font-weight: 800; letter-spacing: -0.4px; margin-bottom: 4px; }
  .section-heading p { font-size: 13.5px; color: var(--muted); margin: 0; }
  .text-button { display: inline-flex; align-items: center; gap: 6px; border: 0; background: transparent; color: var(--teal); font-size: 14px; font-weight: 700; white-space: nowrap; }
  .text-button:hover { color: var(--teal-dark); }

  .table-card { background: #fff; border: 1px solid var(--line); border-radius: 18px; overflow: hidden; }
  .table-header, .table-row {
    display: grid; grid-template-columns: 1.1fr 0.9fr 1.3fr 0.6fr 0.9fr 1.3fr 1.2fr; align-items: center; gap: 12px; padding: 15px 22px;
  }
  .table-header { background: #f6fafc; border-bottom: 1px solid var(--line); color: var(--muted); font-size: 12px; font-weight: 700; }
  .table-row { border-bottom: 1px solid #edf2f6; font-size: 13.5px; color: #3d566d; transition: background 0.15s; }
  .table-row:hover { background: #f8fcfc; }
  .table-row:last-child { border-bottom: 0; }
  .order-id { color: var(--teal); font-weight: 700; overflow-wrap: anywhere; }
  .muted { color: var(--muted); }
  .service-cell { display: flex; align-items: center; gap: 10px; }
  .service-icon { flex: none; width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; background: var(--foam); color: var(--teal); }
  .price-cell { font-weight: 800; color: var(--ink); }
  .badge-stack { display: flex; flex-direction: column; align-items: flex-start; gap: 5px; }
  .row-actions { display: flex; gap: 6px; flex-wrap: wrap; justify-content: flex-end; }

  .status-badge { display: inline-flex; align-items: center; gap: 7px; border-radius: 99px; padding: 5px 11px; font-size: 12px; font-weight: 700; white-space: nowrap; }
  .status-badge.sm { font-size: 11px; padding: 3px 9px; }
  .status-dot { width: 6px; height: 6px; border-radius: 50%; background: currentColor; }
  .pending { background: #fff2e2; color: #b45a09; }
  .processing { background: #e8f1fd; color: #1f56b8; }
  .ready { background: #dff5f4; color: #0a7a7f; }
  .completed { background: #e3f7ee; color: #0b7a4b; }
  .cancelled { background: #fef0ee; color: #b42318; }

  .empty-state { padding: 54px 20px; text-align: center; color: var(--soft); }
  .empty-ico { width: 60px; height: 60px; margin: 0 auto 14px; border-radius: 18px; display: grid; place-items: center; background: var(--foam); color: var(--teal); }
  .empty-state h3 { color: var(--ink); margin: 0 0 6px; font-size: 16px; }
  .empty-state p { margin: 0 0 18px; }

  .orders-toolbar { display: flex; gap: 12px; margin-bottom: 18px; }
  .search-box {
    position: relative; flex: 1; max-width: 420px; height: 46px; display: flex; align-items: center; gap: 10px; padding: 0 14px;
    background: #fff; border: 1.5px solid var(--line); border-radius: 13px; color: var(--soft);
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .search-box:focus-within { border-color: var(--teal); box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
  .search-box input { border: 0; outline: 0; width: 100%; background: transparent; color: var(--ink); font-size: 15px; }
  .search-box input::placeholder { color: #9fb1c0; }

  /* ---------- services ---------- */
  .services-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 18px; }
  .service-card { position: relative; padding: 24px; background: #fff; border: 1px solid var(--line); border-radius: 18px; display: flex; flex-direction: column; }
  .featured-service { border: 2px solid var(--teal); box-shadow: 0 14px 34px rgba(14, 124, 134, 0.14); }
  .large-service-icon { width: 52px; height: 52px; border-radius: 14px; display: grid; place-items: center; margin-bottom: 18px; }
  .blue-bg { background: #e8f1fd; color: #2563c9; }
  .teal-bg { background: var(--foam); color: var(--teal); }
  .orange-bg { background: #fff2e2; color: #c2620a; }
  .service-tag { position: absolute; top: 24px; right: 22px; font-size: 12px; font-weight: 700; color: #2563c9; background: #e8f1fd; border-radius: 99px; padding: 5px 11px; }
  .teal-tag { color: var(--teal-dark); background: var(--foam); }
  .orange-tag { color: #b45a09; background: #fff2e2; }
  .service-card h3 { font-size: 19px; font-weight: 800; letter-spacing: -0.4px; margin-bottom: 8px; }
  .service-card p { font-size: 14px; color: var(--muted); line-height: 1.7; min-height: 72px; }
  .service-price { font-size: 24px; font-weight: 800; letter-spacing: -0.6px; margin: auto 0 18px; }
  .service-price small { font-size: 13px; color: var(--soft); font-weight: 600; letter-spacing: 0; }
  .outline-button { width: 100%; height: 46px; background: #fff; border: 1.5px solid #cfe3e6; border-radius: 13px; color: var(--teal-dark); font-size: 14px; font-weight: 700; transition: all 0.18s; }
  .outline-button:hover { background: var(--teal); border-color: var(--teal); color: #fff; }
  .full-button { width: 100%; }

  .info-banner { margin-top: 22px; padding: 18px 20px; display: flex; align-items: center; gap: 16px; background: var(--foam); border: 1px solid #cdebea; border-radius: 16px; }
  .info-icon { flex: none; width: 42px; height: 42px; border-radius: 12px; display: grid; place-items: center; background: #fff; color: var(--teal); }
  .info-text { flex: 1; min-width: 0; }
  .info-banner strong { font-size: 14.5px; }
  .info-banner p { color: var(--muted); font-size: 13.5px; margin: 4px 0 0; }
  .info-banner > button { border: 0; background: transparent; color: var(--teal-dark); font-size: 14px; font-weight: 800; white-space: nowrap; }

  /* ---------- promo ---------- */
  .code-card { display: flex; align-items: center; gap: 16px; padding: 18px 20px; margin-bottom: 20px; background: #fff; border: 1px solid var(--line); border-radius: 16px; }
  .code-card .info-icon { background: var(--foam); }
  .code-text { flex: 1; min-width: 0; }
  .code-text strong { font-size: 14.5px; }
  .code-text p { color: var(--muted); margin: 4px 0 0; font-size: 13.5px; }
  .code-input { display: flex; gap: 10px; }
  .code-input input { height: 46px; width: 220px; padding: 0 14px; border: 1.5px solid var(--line); border-radius: 13px; text-transform: uppercase; background: #fbfdfe; color: var(--ink); }
  .code-input input:focus { border-color: var(--teal); outline: 0; box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }

  .voucher-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 16px; }
  .voucher-card { display: flex; background: #fff; border: 1px solid var(--line); border-radius: 18px; overflow: hidden; }
  .voucher-value {
    flex: none; width: 104px; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 2px; padding: 14px 8px;
    color: #fff; text-align: center; background: linear-gradient(160deg, var(--teal), #0b5f78); border-right: 2px dashed rgba(255, 255, 255, 0.5);
  }
  .voucher-value strong { font-size: 20px; font-weight: 800; letter-spacing: -0.4px; line-height: 1.15; overflow-wrap: anywhere; }
  .voucher-value small { font-size: 11px; opacity: 0.85; font-weight: 700; }
  .voucher-body { flex: 1; min-width: 0; padding: 16px 18px; display: flex; flex-direction: column; }
  .voucher-body h3 { font-size: 16px; font-weight: 800; margin-bottom: 4px; }
  .voucher-body p { color: var(--muted); font-size: 13px; margin-bottom: 8px; line-height: 1.6; }
  .voucher-body ul { margin: 0 0 12px; padding-left: 18px; color: var(--soft); font-size: 12.5px; line-height: 1.7; }
  .voucher-foot { display: flex; align-items: center; justify-content: space-between; gap: 10px; margin-top: auto; }
  .code-chip { display: inline-block; padding: 5px 11px; border-radius: 8px; background: var(--foam); color: var(--teal-dark); font-size: 12.5px; font-weight: 800; letter-spacing: 0.5px; border: 1px dashed #9fd3d6; }
  .code-chip.clickable { cursor: pointer; }
  .code-chip.clickable:hover { background: #d4efef; }

  /* ---------- loyalty ---------- */
  .loyalty-hero {
    display: flex; justify-content: space-between; align-items: center; gap: 24px; padding: 28px 32px; margin-bottom: 20px;
    border-radius: 20px; color: #fff;
    background: radial-gradient(420px 260px at 95% 0, rgba(43, 196, 192, 0.45), transparent 65%), linear-gradient(120deg, #0c2b45, #0e7c86);
  }
  .lh-label { font-size: 13px; font-weight: 600; color: #9fe3e0; }
  .lh-points { font-size: 52px; font-weight: 800; letter-spacing: -1.5px; line-height: 1.1; margin: 4px 0; }
  .lh-points small { font-size: 18px; font-weight: 600; letter-spacing: 0; color: #c3dbe6; }
  .lh-main p { margin: 0; color: #c3dbe6; font-size: 14px; }
  .lh-main p b { color: #fff; }
  .lh-tier { width: min(300px, 100%); }
  .tier-badge { display: inline-block; margin-bottom: 12px; padding: 6px 14px; border-radius: 99px; background: rgba(255, 255, 255, 0.16); border: 1px solid rgba(255, 255, 255, 0.25); font-weight: 800; font-size: 13px; }
  .lh-tier small { display: block; margin-top: 8px; color: #c3dbe6; font-size: 12.5px; }

  .two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 18px; }
  .panel { padding: 22px 24px; background: #fff; border: 1px solid var(--line); border-radius: 18px; }
  .panel h3 { font-size: 16px; font-weight: 800; margin-bottom: 12px; letter-spacing: -0.3px; }
  .panel h3.mt { margin-top: 22px; }
  .panel .muted { margin: 0; }
  .how-list { margin: 0; padding-left: 18px; color: #3d566d; font-size: 13.5px; line-height: 1.9; }
  .reward-list { display: grid; gap: 8px; }
  .reward-item { display: flex; justify-content: space-between; align-items: center; padding: 11px 14px; border: 1.5px solid var(--line); border-radius: 12px; font-size: 13.5px; color: var(--soft); }
  .reward-item strong { color: var(--ink); }
  .reward-item.ok { border-color: #9fd3d6; background: #f1fafa; color: var(--teal-dark); }
  .history-list { display: grid; }
  .history-item { display: flex; justify-content: space-between; align-items: center; gap: 12px; padding: 12px 0; border-bottom: 1px dashed #e5edf2; }
  .history-item:last-child { border-bottom: 0; }
  .history-item strong { display: block; font-size: 13.5px; color: var(--teal); overflow-wrap: anywhere; }
  .history-item small { color: var(--soft); font-size: 12px; }
  .history-pts { display: flex; flex-direction: column; align-items: flex-end; gap: 2px; font-weight: 800; font-size: 13.5px; }
  .pts.plus { color: #0b7a4b; }
  .pts.minus { color: #b42318; }

  /* ---------- profile ---------- */
  .profile-card { background: #fff; border: 1px solid var(--line); border-radius: 20px; overflow: hidden; }
  .profile-cover { height: 130px; background: radial-gradient(300px 200px at 90% 0, rgba(43, 196, 192, 0.5), transparent 70%), linear-gradient(120deg, #0c2b45, #0e7c86); }
  .profile-main { display: flex; align-items: flex-end; gap: 18px; padding: 0 28px; margin-top: -38px; }
  .profile-avatar { flex: none; width: 84px; height: 84px; border: 5px solid #fff; border-radius: 24px; display: grid; place-items: center; background: var(--foam); color: var(--teal); font-size: 24px; font-weight: 800; }
  .profile-details { padding-bottom: 4px; min-width: 0; }
  .profile-details h2 { font-size: 21px; font-weight: 800; letter-spacing: -0.5px; margin-bottom: 4px; }
  .profile-details p { color: var(--muted); font-size: 14px; margin-bottom: 10px; overflow-wrap: anywhere; }
  .member-badge { display: inline-block; color: var(--teal-dark); background: var(--foam); border-radius: 99px; padding: 5px 12px; font-size: 12px; font-weight: 700; }
  .profile-fields { padding: 28px; display: grid; grid-template-columns: 1fr 1fr; gap: 22px 28px; }
  .pf-label { display: block; font-size: 12.5px; color: var(--soft); font-weight: 600; margin-bottom: 8px; }
  .profile-field div { font-size: 15px; color: var(--ink); font-weight: 700; }
  .wrap-any { overflow-wrap: anywhere; }

  .footer { border-top: 1px solid var(--line); padding: 18px clamp(20px, 3vw, 40px); display: flex; justify-content: space-between; gap: 10px; color: var(--soft); font-size: 12.5px; background: #fff; }

  /* ---------- toast ---------- */
  .toast-zone { position: fixed; top: 18px; right: 18px; z-index: 200; display: grid; gap: 10px; pointer-events: none; }
  .toast { pointer-events: auto; display: flex; align-items: flex-start; gap: 10px; max-width: 360px; padding: 13px 16px; border-radius: 14px; font-size: 14px; line-height: 1.5; font-weight: 600; box-shadow: 0 18px 40px rgba(12, 43, 69, 0.2); animation: pop 0.25s ease; }
  .toast svg { flex: none; margin-top: 1px; }
  .toast.success { background: #eafaf2; color: #0b7a4b; border: 1px solid #b5ead0; }
  .toast.error { background: #fef3f2; color: #b42318; border: 1px solid #fcd0cc; }

  /* ---------- bottom nav & fab (hidden on desktop) ---------- */
  .bottom-nav, .fab { display: none; }

  /* ---------- modal ---------- */
  .modal-overlay { position: fixed; inset: 0; z-index: 100; display: flex; align-items: center; justify-content: center; padding: 20px; background: rgba(8, 34, 56, 0.55); backdrop-filter: blur(3px); animation: fade 0.2s ease; }
  @keyframes fade { from { opacity: 0; } to { opacity: 1; } }
  @keyframes modal-in { from { opacity: 0; transform: translateY(14px) scale(0.98); } to { opacity: 1; transform: none; } }
  .modal { width: 100%; max-width: 600px; max-height: 92vh; max-height: 92dvh; display: flex; flex-direction: column; overflow: hidden; background: #fff; border-radius: 22px; box-shadow: 0 30px 80px rgba(8, 34, 56, 0.35); animation: modal-in 0.25s ease; }
  .modal-header { padding: 24px 26px 18px; border-bottom: 1px solid var(--line); display: flex; justify-content: space-between; align-items: flex-start; }
  .modal-header h2 { font-size: 22px; font-weight: 800; letter-spacing: -0.5px; margin-bottom: 4px; }
  .modal-header p { color: var(--muted); font-size: 13.5px; margin: 0; }
  .close-button { width: 36px; height: 36px; border: 0; border-radius: 10px; display: grid; place-items: center; background: #eef4f8; color: var(--muted); flex: none; }
  .close-button:hover { background: #dfe9f0; }
  .modal-body { padding: 22px 26px; overflow-y: auto; }

  .form-group { margin-bottom: 22px; }
  .form-group > label, .group-label { display: block; color: #243f57; font-size: 13px; font-weight: 700; margin-bottom: 10px; }
  .field-label { display: block; color: #243f57; font-size: 12.5px; font-weight: 700; margin-bottom: 8px; }
  .field-hint { display: block; color: var(--soft); font-size: 12px; margin-top: 8px; }
  .field-input { width: 100%; height: 46px; padding: 0 14px; border: 1.5px solid var(--line); border-radius: 12px; background: #fff; color: var(--ink); font-size: 15px; outline: 0; }
  textarea.field-input { height: auto; padding: 12px 14px; resize: vertical; }
  .field-input:focus { border-color: var(--teal); box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
  .two-fields { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }

  .weight-input { display: flex; align-items: center; height: 50px; border: 1.5px solid var(--line); border-radius: 13px; overflow: hidden; background: #fbfdfe; }
  .weight-input:focus-within { border-color: var(--teal); box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
  .weight-input button { width: 50px; height: 100%; border: 0; background: #f1f7f9; color: var(--teal); font-size: 22px; font-weight: 600; }
  .weight-input button:hover { background: var(--foam); }
  .weight-input input { flex: 1; min-width: 0; height: 100%; border: 0; outline: 0; text-align: center; background: transparent; color: var(--ink); font-size: 17px; font-weight: 800; }
  .kg-label { padding: 0 16px; color: var(--soft); font-size: 12px; font-weight: 700; }

  .option-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .option-grid.three { grid-template-columns: repeat(3, 1fr); }
  .option-card { display: flex; flex-direction: column; align-items: flex-start; gap: 4px; padding: 14px; text-align: left; background: #fff; border: 1.5px solid var(--line); border-radius: 14px; color: var(--soft); transition: border-color 0.18s, background 0.18s; }
  .option-card svg { margin-bottom: 6px; }
  .option-card:hover { border-color: #9fd3d6; }
  .option-card.selected { border-color: var(--teal); background: #f1fafa; box-shadow: 0 0 0 3px rgba(14, 124, 134, 0.1); color: var(--teal); }
  .option-card strong { font-size: 13.5px; color: var(--ink); }
  .option-card small { color: var(--muted); font-size: 12px; }

  .pickup-details { background: #f6fafc; border: 1px solid #e3edf3; border-radius: 16px; padding: 16px; margin-bottom: 22px; }
  .pickup-details .form-group:last-child { margin-bottom: 0; }
  .pickup-details textarea { width: 100%; resize: vertical; border: 1.5px solid var(--line); border-radius: 12px; padding: 12px 14px; background: #fff; outline: 0; color: var(--ink); font-size: 15px; }
  .pickup-details textarea:focus { border-color: var(--teal); box-shadow: 0 0 0 4px rgba(14, 124, 134, 0.12); }
  .required { color: #e5484d; }

  .check-row { display: flex; align-items: flex-start; gap: 10px; margin-bottom: 14px; font-size: 13.5px; color: #3d566d; font-weight: 600; cursor: pointer; }
  .check-row input { width: 18px; height: 18px; margin-top: 1px; accent-color: var(--teal); flex: none; }
  .check-row small { display: block; color: var(--soft); font-weight: 500; font-size: 12.5px; margin-top: 2px; }
  .points-row { padding: 14px; margin-bottom: 22px; background: #f6f2ff; border: 1px solid #e3d9fb; border-radius: 14px; }

  .voucher-input { display: flex; gap: 10px; }
  .voucher-input .field-input { text-transform: uppercase; }
  .voucher-input .outline-button.apply { width: 96px; flex: none; height: 46px; }
  .voucher-input .cancel-button { flex: none; }
  .voucher-msg { display: block; margin-top: 8px; font-size: 12.5px; color: #b42318; font-weight: 600; }
  .voucher-msg.ok { color: #0b7a4b; }
  .chip-row { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 10px; }

  .upload-box { position: relative; height: 128px; display: flex; align-items: center; justify-content: center; overflow: hidden; background: #fff; border: 1.5px dashed #b9d3d8; border-radius: 14px; cursor: pointer; transition: border-color 0.18s, background 0.18s; }
  .upload-box:hover { border-color: var(--teal); background: #f6fcfc; }
  .upload-content { text-align: center; padding: 0 12px; }
  .upload-icon { width: 38px; height: 38px; margin: 0 auto 8px; border-radius: 11px; display: grid; place-items: center; background: var(--foam); color: var(--teal); }
  .upload-content strong { display: block; color: var(--teal-dark); font-size: 13px; }
  .upload-content small { display: block; color: var(--soft); font-size: 12px; margin-top: 4px; }
  .upload-box img { width: 100%; height: 100%; object-fit: cover; }
  .upload-overlay { position: absolute; inset: 0; display: grid; place-items: center; background: rgba(8, 34, 56, 0.55); color: #fff; font-size: 13px; font-weight: 700; opacity: 0; transition: opacity 0.2s; }
  .upload-box:hover .upload-overlay { opacity: 1; }
  .uploaded-name { display: block; color: #0b7a4b; font-size: 12px; margin-top: 8px; }

  .price-summary { background: #f6fafc; border: 1px solid #e3edf3; border-radius: 16px; padding: 16px; }
  .summary-title { display: flex; justify-content: space-between; margin-bottom: 14px; }
  .summary-title strong { font-size: 14px; }
  .summary-title span { color: var(--soft); font-size: 12px; }
  .summary-row { display: flex; justify-content: space-between; gap: 12px; margin-bottom: 9px; color: var(--muted); font-size: 13.5px; }
  .summary-row span:last-child { color: var(--ink); font-weight: 600; white-space: nowrap; }
  .summary-row.discount span:last-child { color: #0b7a4b; }
  .summary-total { display: flex; justify-content: space-between; align-items: center; margin-top: 12px; padding-top: 14px; border-top: 1px dashed #cfdde6; }
  .summary-total span { font-size: 14px; color: var(--muted); font-weight: 600; }
  .summary-total strong { font-size: 22px; font-weight: 800; letter-spacing: -0.5px; color: var(--teal); }
  .earn-hint { margin-top: 10px; font-size: 12.5px; color: #6a45d1; }

  .modal-footer { padding: 16px 26px; border-top: 1px solid var(--line); display: flex; justify-content: flex-end; flex-wrap: wrap; gap: 10px; }
  .cancel-button { height: 46px; padding: 0 20px; border: 0; border-radius: 13px; background: #eef4f8; color: #3d566d; font-size: 14px; font-weight: 700; }
  .cancel-button:hover { background: #e1ebf1; }

  /* ---------- detail / receipt ---------- */
  .detail-badges { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 18px; }
  .detail-block { margin-bottom: 20px; }
  .detail-block h4 { margin: 0 0 10px; font-size: 14px; font-weight: 800; }
  .detail-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; padding: 14px; background: #f6fafc; border: 1px solid #e3edf3; border-radius: 14px; }
  .detail-grid b { display: block; font-size: 13.5px; }
  .detail-grid p { margin: 4px 0 0; color: var(--muted); font-size: 12.5px; line-height: 1.6; overflow-wrap: anywhere; }
  .detail-note { margin: 10px 0 0; color: var(--muted); font-size: 13px; }
  .receipt { padding: 16px; background: #fff; border: 1.5px dashed #cfdde6; border-radius: 14px; }
  .receipt-meta { display: grid; gap: 4px; margin-top: 12px; padding-top: 12px; border-top: 1px dashed #cfdde6; color: var(--muted); font-size: 12.5px; }
  .receipt-brand { margin-bottom: 14px; }
  .stars-static { display: flex; gap: 2px; color: var(--gold); }

  /* ---------- pembayaran ---------- */
  .pay-box { padding: 16px; margin-bottom: 20px; background: #f6fafc; border: 1px solid #e3edf3; border-radius: 16px; }
  .pay-box p { margin: 12px 0 0; color: var(--muted); font-size: 13.5px; line-height: 1.65; }
  .pay-box p:first-child { margin-top: 0; }
  .qris-img { display: block; width: 200px; max-width: 100%; margin: 0 auto; border-radius: 12px; background: #fff; }
  .qris-placeholder { display: grid; place-items: center; gap: 8px; height: 170px; border: 1.5px dashed #b9d3d8; border-radius: 14px; background: #fff; color: var(--teal); text-align: center; padding: 10px; }
  .qris-placeholder small { color: var(--soft); font-size: 12px; }
  .bank-row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 12px 14px; margin-bottom: 8px; background: #fff; border: 1px solid var(--line); border-radius: 12px; }
  .bank-row strong { display: block; font-size: 13px; color: var(--teal-dark); }
  .bank-row span { display: block; font-size: 17px; font-weight: 800; letter-spacing: 1px; }
  .bank-row small { color: var(--soft); font-size: 12px; }

  /* ---------- rating ---------- */
  .rate-row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 10px 0; border-bottom: 1px dashed #e5edf2; }
  .rate-row .group-label { margin: 0; }
  .rate-row.big { flex-direction: column; align-items: center; padding: 6px 0 18px; text-align: center; }
  .rate-row.big .group-label { font-size: 14px; margin-bottom: 8px; }
  .stars { display: flex; gap: 2px; }
  .stars button { border: 0; background: transparent; padding: 2px; color: #c9d6df; transition: transform 0.12s, color 0.12s; }
  .stars button:hover { transform: scale(1.12); }
  .stars button.on { color: var(--gold); }
  .rate-row + .form-group, .rate-row:last-of-type { margin-bottom: 4px; }
  .modal-body > .form-group:last-child { margin-top: 18px; margin-bottom: 0; }

  /* ================= RESPONSIVE ================= */
  @media (max-width: 1200px) {
    .stats-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
    .stats-grid.three { grid-template-columns: repeat(3, minmax(0, 1fr)); }
    .hero-content { width: 66%; }
    .action-row, .two-col { grid-template-columns: 1fr; }
  }

  /* Tablet & HP: sidebar diganti navigasi bawah */
  @media (max-width: 900px) {
    .sidebar { display: none; }
    .mobile-brand { display: flex; }
    .breadcrumb, .date-chip { display: none; }
    .mobile-only { display: grid; }
    .topbar { height: 62px; padding-top: env(safe-area-inset-top); }

    .main-content { padding-bottom: calc(76px + env(safe-area-inset-bottom)); }

    .bottom-nav {
      position: fixed; left: 0; right: 0; bottom: 0; z-index: 30;
      display: grid; grid-template-columns: repeat(6, 1fr);
      padding: 6px 6px calc(6px + env(safe-area-inset-bottom));
      background: rgba(255, 255, 255, 0.95); backdrop-filter: blur(12px);
      border-top: 1px solid var(--line);
    }
    .bottom-nav button {
      display: flex; flex-direction: column; align-items: center; gap: 4px; padding: 8px 2px;
      border: 0; background: transparent; color: var(--soft); font-size: 10.5px; font-weight: 700; border-radius: 12px;
    }
    .bottom-nav button.active { color: var(--teal); background: var(--foam); }

    .fab {
      position: fixed; right: 18px; bottom: calc(86px + env(safe-area-inset-bottom)); z-index: 31;
      width: 56px; height: 56px; display: grid; place-items: center; border: 0; border-radius: 18px;
      color: #fff; background: linear-gradient(135deg, var(--teal), #0b5f78); box-shadow: 0 14px 28px rgba(14, 124, 134, 0.4);
    }

    .services-grid { grid-template-columns: 1fr; }
    .service-card p { min-height: 0; }
    .hero-content { width: 100%; }
    .hero-decoration { opacity: 0.35; width: 55%; }
    .footer { flex-direction: column; }
    .code-card { flex-wrap: wrap; }
    .code-input { width: 100%; }
    .code-input input { flex: 1; width: auto; }
    .loyalty-hero { flex-direction: column; align-items: flex-start; }
  }

  /* HP: tabel jadi kartu */
  @media (max-width: 760px) {
    .page-content { padding: 20px 16px; }
    .welcome-section { flex-direction: column; align-items: stretch; }
    .welcome-section > .primary-button { width: 100%; }
    h1 { font-size: 24px; }

    .stats-grid, .stats-grid.three { gap: 10px; grid-template-columns: repeat(2, minmax(0, 1fr)); }
    .stat-card { flex-direction: column; align-items: flex-start; padding: 14px; gap: 10px; }
    .stat-icon { width: 40px; height: 40px; }
    .stat-card h2 { font-size: 22px; }
    .lh-points { font-size: 42px; }
    .loyalty-hero { padding: 22px 20px; }

    .hero-banner { min-height: 250px; }
    .hero-content { padding: 26px 22px; }
    .hero-content h2 { font-size: 22px; }
    .hero-content p { max-width: 250px; }
    .hero-decoration { opacity: 0.22; width: 70%; }
    .floating-card { display: none; }

    .table-card { background: transparent; border: 0; border-radius: 0; overflow: visible; display: grid; gap: 12px; }
    .table-header { display: none; }
    .table-row {
      display: block; padding: 6px 16px 10px; background: #fff; border: 1px solid var(--line); border-radius: 16px;
    }
    .table-row:last-child { border-bottom: 1px solid var(--line); }
    .table-row > div {
      display: flex; justify-content: space-between; align-items: center; gap: 14px;
      padding: 10px 0; border-bottom: 1px dashed #e5edf2; text-align: right;
    }
    .table-row > div:last-child { border-bottom: 0; }
    .table-row > div::before { content: attr(data-label); flex: none; color: var(--soft); font-size: 12px; font-weight: 600; text-align: left; }
    .badge-stack { align-items: flex-end; }
    .table-card > .empty-state { background: #fff; border: 1px solid var(--line); border-radius: 16px; }

    .orders-toolbar .search-box { max-width: none; }
    .profile-fields { grid-template-columns: 1fr; padding: 22px 20px; }
    .profile-main { padding: 0 20px; }
    .info-banner { flex-wrap: wrap; }
    .info-text { flex-basis: calc(100% - 70px); }
    .info-banner > button { margin-left: 58px; padding: 0; }
    .voucher-grid { grid-template-columns: 1fr; }
    .detail-grid { grid-template-columns: 1fr; }

    .toast-zone { left: 16px; right: 16px; top: calc(10px + env(safe-area-inset-top)); }
    .toast { max-width: none; }

    .modal-overlay { align-items: flex-end; padding: 0; }
    .modal { max-width: none; max-height: 94dvh; border-radius: 24px 24px 0 0; animation: sheet 0.28s ease; }
    .modal-header, .modal-body, .modal-footer { padding-left: 18px; padding-right: 18px; }
    .modal-footer { padding-bottom: calc(16px + env(safe-area-inset-bottom)); flex-direction: column-reverse; }
    .modal-footer > * { width: 100%; }
    .option-grid.three { grid-template-columns: 1fr; }
    .option-grid.three .option-card { flex-direction: row; align-items: center; gap: 10px; }
    .option-grid.three .option-card svg { margin: 0; flex: none; }
    .two-fields { grid-template-columns: 1fr; }
  }
  @keyframes sheet { from { transform: translateY(40px); opacity: 0; } to { transform: none; opacity: 1; } }

  @media (max-width: 380px) {
    .option-grid { grid-template-columns: 1fr; }
    .stat-label { font-size: 12px; }
    .bottom-nav button { font-size: 9.5px; }
  }

  @media (prefers-reduced-motion: reduce) {
    * { animation: none !important; transition: none !important; }
  }

  /* ---------- cetak bukti transaksi ---------- */
  @media print {
    :global(body) { background: #fff; }
    .app-shell, .toast-zone, .no-print { display: none !important; }
    .print-only { display: flex; flex-direction: column; gap: 2px; }
    .receipt-brand strong { font-size: 20px; }
    .modal-overlay { position: static; display: block; padding: 0; background: none; backdrop-filter: none; }
    .modal { max-width: none; max-height: none; box-shadow: none; border-radius: 0; animation: none; }
    .modal-body { overflow: visible; padding: 0; }
  }
</style>