<script lang="ts">
  import { supabase } from "$lib/supabaseClient";
  import { onMount } from "svelte";
  import * as XLSX from "xlsx";

  let users: { id: number; name: string }[] = [];
  let absenData: any[] = [];

  let sholat = "Subuh";
  let tanggal = new Date().toISOString().split("T")[0];
  let activeSection: "form" | "rekap" = "form";

  let editingId: number | null = null;
  let editStatus: string = "";

  onMount(async () => {
    const { data, error } = await supabase.from("users").select("*").order("id");
    if (!error) users = data;
    await loadAbsen();
  });

  async function loadAbsen() {
    const { data, error } = await supabase
      .from("absen")
      .select("id, user_id, status, tanggal, sholat, users(name)")
      .eq("tanggal", tanggal)
      .eq("sholat", sholat);

    absenData = error ? [] : data;
    editingId = null;
  }

  async function submitAbsen(e: Event) {
    e.preventDefault();
    const formData = new FormData(e.target as HTMLFormElement);

    for (let user of users) {
      const status = formData.get(`${user.id}-status`) as string;
      if (!status) continue;

      const { data: existing } = await supabase
        .from("absen")
        .select("id")
        .eq("user_id", user.id)
        .eq("sholat", sholat)
        .eq("tanggal", tanggal)
        .maybeSingle();

      if (existing) {
        await supabase.from("absen").update({ status }).eq("id", existing.id);
      } else {
        await supabase.from("absen").insert({
          user_id: user.id,
          sholat,
          status,
          tanggal
        });
      }
    }

    alert(`✅ Absen ${sholat} tanggal ${tanggal} berhasil disimpan`);
    await loadAbsen();
  }

  async function removeAbsen(id: number) {
    if (!confirm("Apakah yakin ingin menghapus data ini?")) return;
    await supabase.from("absen").delete().eq("id", id);
    await loadAbsen();
  }

  function startEdit(row: any) {
    editingId = row.id;
    editStatus = row.status;
  }

  async function saveEdit(id: number) {
    await supabase.from("absen").update({ status: editStatus }).eq("id", id);
    editingId = null;
    await loadAbsen();
  }

  function exportToExcel() {
    // Format data untuk Excel
    const exportData = absenData.map((row) => ({
      Nama: row.users.name,
      Sholat: row.sholat,
      Tanggal: row.tanggal,
      Status: row.status === "hadir" ? "Hadir" : "Tidak"
    }));

    const worksheet = XLSX.utils.json_to_sheet(exportData);
    const workbook = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(workbook, worksheet, "Rekap Absen");

    XLSX.writeFile(
      workbook,
      `Rekap_Absen_${sholat}_${tanggal}.xlsx`
    );
  }
</script>

<div class="container">
  <h1>📿 Absen Sholat</h1>

  <!-- Tombol navigasi -->
  <div class="section-buttons">
    <button
      class:active={activeSection === "form"}
      on:click={() => (activeSection = "form")}
    >
      Form Absen
    </button>
    <button
      class:active={activeSection === "rekap"}
      on:click={() => (activeSection = "rekap")}
    >
      Rekap Absen
    </button>
  </div>

  <!-- Filter -->
  <div class="filters">
    <div>
      <label>Pilih Sholat:</label>
      <select bind:value={sholat} on:change={loadAbsen}>
        <option value="Subuh">Subuh</option>
        <option value="Dzuhur">Dzuhur</option>
        <option value="Ashar">Ashar</option>
        <option value="Maghrib">Maghrib</option>
        <option value="Isya">Isya</option>
      </select>
    </div>

    <div>
      <label>Tanggal:</label>
      <input type="date" bind:value={tanggal} on:change={loadAbsen} />
    </div>
  </div>

  <!-- Form Absen -->
  {#if activeSection === "form"}
    <form on:submit={submitAbsen} class="form-card">
      {#each users as user}
        <div class="form-row">
          <span>{user.name}</span>
          <select name="{user.id}-status">
            <option value="hadir">✅ Hadir</option>
            <option value="tidak">❌ Tidak</option>
          </select>
        </div>
      {/each}

      <button type="submit">Simpan Absen {sholat} ({tanggal})</button>
    </form>
  {/if}

  <!-- Rekap Absen -->
  {#if activeSection === "rekap"}
    <div class="table-wrapper">
      <div class="export-wrapper">
        <button class="export" on:click={exportToExcel}>⬇️ Export ke Excel</button>
      </div>

      {#if absenData.length > 0}
        <table>
          <thead>
            <tr>
              <th>Nama</th>
              <th>Status</th>
              <th>Aksi</th>
            </tr>
          </thead>
          <tbody>
            {#each absenData as row}
              <tr>
                <td>{row.users.name}</td>
                <td>
                  {#if editingId === row.id}
                    <select bind:value={editStatus}>
                      <option value="hadir">✅ Hadir</option>
                      <option value="tidak">❌ Tidak</option>
                    </select>
                  {:else}
                    {row.status === "hadir" ? "✅ Hadir" : "❌ Tidak"}
                  {/if}
                </td>
                <td>
                  {#if editingId === row.id}
                    <button class="save" on:click={() => saveEdit(row.id)}>Simpan</button>
                    <button class="cancel" on:click={() => (editingId = null)}>Batal</button>
                  {:else}
                    <button class="edit" on:click={() => startEdit(row)}>Edit</button>
                    <button class="delete" on:click={() => removeAbsen(row.id)}>Hapus</button>
                  {/if}
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      {:else}
        <p class="no-data">Belum ada absen {sholat} untuk tanggal {tanggal}</p>
      {/if}
    </div>
  {/if}
</div>

<style>
  :root {
    --primary: #2563eb;
    --primary-hover: #1d4ed8;
    --danger: #dc2626;
    --danger-hover: #b91c1c;
    --border: #e5e7eb;
    --bg: #f9fafb;
  }

  .container {
    max-width: 900px;
    margin: auto;
    padding: 1rem;
  }

  h1 {
    text-align: center;
    font-size: 1.8rem;
    margin-bottom: 1rem;
  }

  .section-buttons {
    display: flex;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }

  .section-buttons button {
    padding: 0.6rem 1.2rem;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: white;
    cursor: pointer;
    font-weight: 600;
    transition: all 0.2s;
  }
  .section-buttons button:hover {
    background: #f3f4f6;
  }
  .section-buttons button.active {
    background: var(--primary);
    color: white;
    border-color: var(--primary);
  }

  .filters {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }
  .filters label {
    font-weight: 600;
    margin-right: 0.5rem;
  }
  .filters select,
  .filters input[type="date"] {
    padding: 0.4rem 0.6rem;
    border: 1px solid var(--border);
    border-radius: 6px;
  }
  @media (min-width: 600px) {
    .filters {
      flex-direction: row;
      justify-content: space-between;
    }
  }

  .form-card {
    background: #fff;
    border-radius: 12px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
    padding: 1.5rem;
  }
  .form-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--border);
    padding: 0.5rem 0;
  }
  .form-row span {
    font-weight: 500;
  }
  .form-row select {
    padding: 0.3rem 0.6rem;
    border: 1px solid var(--border);
    border-radius: 6px;
  }
  .form-card button {
    width: 100%;
    margin-top: 1rem;
    padding: 0.6rem;
    background: var(--primary);
    color: white;
    border: none;
    border-radius: 8px;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.2s;
  }
  .form-card button:hover {
    background: var(--primary-hover);
  }

  .table-wrapper {
    overflow-x: auto;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 1rem;
    border-radius: 8px;
    overflow: hidden;
  }
  thead {
    background: var(--bg);
  }
  th,
  td {
    padding: 0.6rem;
    border: 1px solid var(--border);
    text-align: center;
  }
  tbody tr:nth-child(even) {
    background: #f3f4f6;
  }
  tbody tr:hover {
    background: #e0e7ff;
  }

  .edit {
    background: #10b981;
    color: white;
    border: none;
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    cursor: pointer;
    margin-right: 0.3rem;
  }
  .edit:hover {
    background: #059669;
  }

  .save {
    background: var(--primary);
    color: white;
    border: none;
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    cursor: pointer;
    margin-right: 0.3rem;
  }
  .save:hover {
    background: var(--primary-hover);
  }

  .cancel {
    background: #6b7280;
    color: white;
    border: none;
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    cursor: pointer;
  }
  .cancel:hover {
    background: #4b5563;
  }

  .delete {
    background: var(--danger);
    color: white;
    border: none;
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.2s;
  }
  .delete:hover {
    background: var(--danger-hover);
  }

  .no-data {
    text-align: center;
    margin-top: 1rem;
    color: #6b7280;
  }
  /* ... CSS sama seperti sebelumnya ... */

.export-wrapper {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 0.5rem;
}

.export {
  background: #059669;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
}
.export:hover {
  background: #047857;
}
</style>
