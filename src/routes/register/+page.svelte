<script lang="ts">
  import { supabase } from '$lib/supabaseClient.js';
  import { goto } from "$app/navigation";

  let email = "";
  let password = "";
  let loading = false;
  let message = "";

  async function register(e: Event) {
    e.preventDefault();
    loading = true;
    message = "";
    const { error } = await supabase.auth.signUp({ email, password });

    if (error) {
      message = "❌ " + error.message;
    } else {
      message = "✅ Registrasi berhasil! Silakan login.";
      setTimeout(() => goto("/admin/login"), 2000);
    }
    loading = false;
  }
</script>

<main class="auth-container">
  <h1>🔑 Register Admin</h1>
  <form on:submit={register}>
    <input type="email" placeholder="Email" bind:value={email} required />
    <input type="password" placeholder="Password" bind:value={password} required />
    <button type="submit" disabled={loading}>{loading ? "Mendaftarkan..." : "Daftar"}</button>
  </form>
  <p>Sudah punya akun? <a href="/admin/login">Login di sini</a></p>
  <p>{message}</p>
</main>

<style>
  .auth-container {
    max-width: 400px;
    margin: 5rem auto;
    background: white;
    padding: 2rem;
    border-radius: 12px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    text-align: center;
  }
  input {
    display: block;
    width: 100%;
    margin-bottom: 1rem;
    padding: 0.8rem;
    border-radius: 8px;
    border: 1px solid #d1d5db;
  }
  button {
    background: #2563eb;
    color: white;
    border: none;
    padding: 0.8rem 1.2rem;
    border-radius: 8px;
    cursor: pointer;
    width: 100%;
  }
  button:hover { background: #1d4ed8; }
</style>
