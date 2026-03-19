<script>
  import { onMount } from 'svelte';

  const API_BASE = 'https://d1-template.rapid-band-96d6.workers.dev/api';
  const TOKEN_KEY = 'krak_admin_token';

  // AUTH
  let isAuthenticated = false;
  let loginEmail = '';
  let loginPassword = '';
  let loginError = '';
  let isLoggingIn = false;

  // TABS
  let activeTab = 'postulados'; // 'postulados' | 'videos'

  // DATA
  let postulados = [];
  let videos = [];
  let isLoadingPostulados = false;
  let isLoadingVideos = false;

  // NEW VIDEO FORM
  let showNewVideoForm = false;
  let newVideo = {
    link: '',
    expiration_at: '',
    active: 1
  };
  let isCreatingVideo = false;
  let createVideoError = '';

  // EDIT VIDEO
  let showEditVideoForm = false;
  let editingVideo = null;
  let isEditingVideo = false;
  let editVideoError = '';

  // DELETE VIDEO
  let showDeleteConfirm = false;
  let deletingVideoId = null;
  let isDeletingVideo = false;

  onMount(() => {
    const token = localStorage.getItem(TOKEN_KEY);
    if (token) {
      isAuthenticated = true;
      loadData();
    }
  });

  async function handleLogin(e) {
    e.preventDefault();
    loginError = '';
    isLoggingIn = true;

    try {
      const res = await fetch(`${API_BASE}/auth/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email: loginEmail,
          password: loginPassword
        })
      });

      if (!res.ok) {
        loginError = 'Credenciales incorrectas';
        return;
      }

      const data = await res.json();

      if (data.token) {
        localStorage.setItem(TOKEN_KEY, data.token);
        isAuthenticated = true;
        loadData();
      } else {
        loginError = 'Error en la respuesta del servidor';
      }
    } catch (error) {
      console.error(error);
      loginError = 'Error de conexión';
    } finally {
      isLoggingIn = false;
    }
  }

  function handleLogout() {
    localStorage.removeItem(TOKEN_KEY);
    isAuthenticated = false;
    loginEmail = '';
    loginPassword = '';
    postulados = [];
    videos = [];
  }

  async function loadData() {
    if (activeTab === 'postulados') {
      await loadPostulados();
    } else {
      await loadVideos();
    }
  }

  async function loadPostulados() {
    isLoadingPostulados = true;
    try {
      const token = localStorage.getItem(TOKEN_KEY);
      const res = await fetch(`${API_BASE}/formularios`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });

      if (res.ok) {
        const rawData = await res.json();
        const list = Array.isArray(rawData) ? rawData : rawData.formularios || [];

        // Parse data field (JSON string) y combinar con file
        postulados = list.map(item => {
          let parsed = {};
          try {
            parsed = JSON.parse(item.data || '{}');
          } catch (e) {
            console.error('Error parsing data:', e);
          }

          return {
            id: item.id,
            created_at: item.created_at,
            nombre: parsed.nombre || '',
            apellido: parsed.apellido || '',
            email: parsed.email || '',
            celular: parsed.celular || '',
            file: item.file || ''
          };
        });
      } else if (res.status === 401) {
        handleLogout();
      }
    } catch (error) {
      console.error(error);
    } finally {
      isLoadingPostulados = false;
    }
  }

  async function loadVideos() {
    isLoadingVideos = true;
    try {
      const token = localStorage.getItem(TOKEN_KEY);
      const res = await fetch(`${API_BASE}/videos`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });

      if (res.ok) {
        const data = await res.json();
        videos = Array.isArray(data) ? data : data.videos || [];
      } else if (res.status === 401) {
        handleLogout();
      }
    } catch (error) {
      console.error(error);
    } finally {
      isLoadingVideos = false;
    }
  }

  function switchTab(tab) {
    activeTab = tab;
    loadData();
  }

  function formatDate(isoString) {
    if (!isoString) return '-';
    const d = new Date(isoString);
    return d.toLocaleDateString('es-AR', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit'
    });
  }

  function openNewVideoForm() {
    showNewVideoForm = true;
    newVideo = {
      link: '',
      expiration_at: '',
      active: 1
    };
    createVideoError = '';
  }

  function closeNewVideoForm() {
    showNewVideoForm = false;
    newVideo = {
      link: '',
      expiration_at: '',
      active: 1
    };
    createVideoError = '';
  }

  async function handleCreateVideo(e) {
    e.preventDefault();
    createVideoError = '';
    isCreatingVideo = true;

    try {
      const token = localStorage.getItem(TOKEN_KEY);
      const res = await fetch(`${API_BASE}/videos`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          link: newVideo.link.trim(),
          expiration_at: newVideo.expiration_at,
          active: newVideo.active
        })
      });

      if (res.ok) {
        closeNewVideoForm();
        await loadVideos();
      } else if (res.status === 401) {
        handleLogout();
      } else {
        const errorData = await res.json().catch(() => ({}));
        createVideoError = errorData.message || 'Error al crear el video';
      }
    } catch (error) {
      console.error(error);
      createVideoError = 'Error de conexión';
    } finally {
      isCreatingVideo = false;
    }
  }

  function openEditVideoForm(video) {
    editingVideo = {
      id: video.id,
      link: video.link || '',
      expiration_at: video.expiration_at ? formatDateTimeLocal(video.expiration_at) : '',
      active: video.active ?? 1
    };
    showEditVideoForm = true;
    editVideoError = '';
  }

  function closeEditVideoForm() {
    showEditVideoForm = false;
    editingVideo = null;
    editVideoError = '';
  }

  async function handleEditVideo(e) {
    e.preventDefault();
    editVideoError = '';
    isEditingVideo = true;

    try {
      const token = localStorage.getItem(TOKEN_KEY);
      const res = await fetch(`${API_BASE}/videos/${editingVideo.id}`, {
        method: 'PUT',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          link: editingVideo.link.trim(),
          expiration_at: editingVideo.expiration_at,
          active: editingVideo.active
        })
      });

      if (res.ok) {
        closeEditVideoForm();
        await loadVideos();
      } else if (res.status === 401) {
        handleLogout();
      } else {
        const errorData = await res.json().catch(() => ({}));
        editVideoError = errorData.message || 'Error al actualizar el video';
      }
    } catch (error) {
      console.error(error);
      editVideoError = 'Error de conexión';
    } finally {
      isEditingVideo = false;
    }
  }

  function openDeleteConfirm(videoId) {
    deletingVideoId = videoId;
    showDeleteConfirm = true;
  }

  function closeDeleteConfirm() {
    showDeleteConfirm = false;
    deletingVideoId = null;
  }

  async function handleDeleteVideo() {
    isDeletingVideo = true;

    try {
      const token = localStorage.getItem(TOKEN_KEY);
      const res = await fetch(`${API_BASE}/videos/${deletingVideoId}`, {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });

      if (res.ok) {
        closeDeleteConfirm();
        await loadVideos();
      } else if (res.status === 401) {
        handleLogout();
      } else {
        alert('Error al eliminar el video');
      }
    } catch (error) {
      console.error(error);
      alert('Error de conexión');
    } finally {
      isDeletingVideo = false;
    }
  }

  function formatDateTimeLocal(isoString) {
    if (!isoString) return '';
    const d = new Date(isoString);
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    const hours = String(d.getHours()).padStart(2, '0');
    const minutes = String(d.getMinutes()).padStart(2, '0');
    return `${year}-${month}-${day}T${hours}:${minutes}`;
  }

  function extractYouTubeId(url) {
    if (!url) return null;
    const patterns = [
      /(?:youtube\.com\/watch\?v=|youtu\.be\/)([^&\s]+)/,
      /youtube\.com\/embed\/([^?&\s]+)/,
      /youtube\.com\/v\/([^?&\s]+)/
    ];
    for (const pattern of patterns) {
      const match = url.match(pattern);
      if (match) return match[1];
    }
    return null;
  }
</script>

<svelte:head>
  <title>Admin - Krak Agentes</title>
</svelte:head>

<div class="min-h-screen bg-[#050A14]">
  {#if !isAuthenticated}
    <!-- LOGIN -->
    <div class="flex items-center justify-center min-h-screen px-4">
      <div class="w-full max-w-md">
        <div class="rounded-2xl border border-white/10 bg-white/5 p-8 text-white">
          <h1 class="text-3xl font-bold text-center mb-6">Admin Login</h1>

          <form on:submit={handleLogin} class="space-y-5">
            <div>
              <label class="block text-sm text-white/70 mb-2">Email</label>
              <input
                type="email"
                bind:value={loginEmail}
                required
                class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
                placeholder="admin@krak.com.ar"
              />
            </div>

            <div>
              <label class="block text-sm text-white/70 mb-2">Password</label>
              <input
                type="password"
                bind:value={loginPassword}
                required
                class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
              />
            </div>

            {#if loginError}
              <p class="text-red-400 text-sm">{loginError}</p>
            {/if}

            <button
              type="submit"
              disabled={isLoggingIn}
              class="w-full py-3 rounded-xl bg-[#08407C] hover:bg-[#0A4D95] text-white font-bold transition disabled:opacity-50"
            >
              {isLoggingIn ? 'Ingresando...' : 'Ingresar'}
            </button>
          </form>
        </div>
      </div>
    </div>
  {:else}
    <!-- ADMIN PANEL -->
    <div class="max-w-7xl mx-auto px-4 py-8">
      <!-- HEADER -->
      <div class="flex items-center justify-between mb-[70px] mt-[10px]">
        <h1 class="text-3xl font-bold text-white">Panel Admin</h1>
        <button
          on:click={handleLogout}
          class="px-4 py-2 rounded-lg bg-white/10 hover:bg-white/20 text-white text-sm transition"
        >
          Cerrar sesión
        </button>
      </div>

      <!-- TABS -->
      <div class="flex gap-2 mb-6 border-b border-white/10">
        <button
          on:click={() => switchTab('postulados')}
          class="px-6 py-3 text-white font-semibold transition {activeTab === 'postulados'
            ? 'border-b-2 border-[#08407C]'
            : 'text-white/60 hover:text-white/90'}"
        >
          Postulados
        </button>

        <button
          on:click={() => switchTab('videos')}
          class="px-6 py-3 text-white font-semibold transition {activeTab === 'videos'
            ? 'border-b-2 border-[#08407C]'
            : 'text-white/60 hover:text-white/90'}"
        >
          Videos
        </button>
      </div>

      <!-- CONTENT -->
      {#if activeTab === 'postulados'}
        <div class="rounded-2xl border border-white/10 bg-white/5 overflow-hidden">
          <div class="p-6">
            <div class="flex items-center justify-between mb-4">
              <h2 class="text-xl font-bold text-white">Postulados para Agentes</h2>
              <button
                on:click={loadPostulados}
                class="px-4 py-2 rounded-lg bg-white/10 hover:bg-white/20 text-white text-sm transition"
              >
                Actualizar
              </button>
            </div>

            {#if isLoadingPostulados}
              <p class="text-white/60 text-center py-8">Cargando...</p>
            {:else if postulados.length === 0}
              <p class="text-white/60 text-center py-8">No hay postulados</p>
            {:else}
              <div class="overflow-x-auto">
                <table class="w-full text-left">
                  <thead>
                    <tr class="border-b border-white/10">
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Fecha</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Nombre</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Email</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Celular</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">CV</th>
                    </tr>
                  </thead>
                  <tbody>
                    {#each postulados as postulado}
                      <tr class="border-b border-white/5 hover:bg-white/5 transition">
                        <td class="px-4 py-3 text-white/90 text-sm">{formatDate(postulado.created_at)}</td>
                        <td class="px-4 py-3 text-white text-sm font-medium">
                          {postulado.nombre} {postulado.apellido}
                        </td>
                        <td class="px-4 py-3 text-white/90 text-sm">{postulado.email}</td>
                        <td class="px-4 py-3 text-white/90 text-sm">{postulado.celular}</td>
                        <td class="px-4 py-3 text-white/90 text-sm">
                          {#if postulado.file}
                            <a
                              href="https://services.franco.in.net/api/krak-files/get?token={postulado.file}"
                              target="_blank"
                              rel="noopener noreferrer"
                              class="text-[#08407C] hover:text-[#0A4D95] underline"
                            >
                              Ver CV
                            </a>
                          {:else}
                            -
                          {/if}
                        </td>
                      </tr>
                    {/each}
                  </tbody>
                </table>
              </div>
            {/if}
          </div>
        </div>
      {:else}
        <div class="rounded-2xl border border-white/10 bg-white/5 overflow-hidden">
          <div class="p-6">
            <div class="flex items-center justify-between mb-4">
              <h2 class="text-xl font-bold text-white">Videos Subidos</h2>
              <div class="flex gap-2">
                <button
                  on:click={openNewVideoForm}
                  class="px-4 py-2 rounded-lg bg-[#08407C] hover:bg-[#0A4D95] text-white text-sm transition font-semibold"
                >
                  + Nuevo Video
                </button>
                <button
                  on:click={loadVideos}
                  class="px-4 py-2 rounded-lg bg-white/10 hover:bg-white/20 text-white text-sm transition"
                >
                  Actualizar
                </button>
              </div>
            </div>

            {#if isLoadingVideos}
              <p class="text-white/60 text-center py-8">Cargando...</p>
            {:else if videos.length === 0}
              <p class="text-white/60 text-center py-8">No hay videos</p>
            {:else}
              <div class="overflow-x-auto">
                <table class="w-full text-left">
                  <thead>
                    <tr class="border-b border-white/10">
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Preview</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Link</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Expira</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Activo</th>
                      <th class="px-4 py-3 text-white/70 text-sm font-semibold">Acciones</th>
                    </tr>
                  </thead>
                  <tbody>
                    {#each videos as video}
                      <tr class="border-b border-white/5 hover:bg-white/5 transition">
                        <td class="px-4 py-3">
                          {#if extractYouTubeId(video.link)}
                            <a href={video.link} target="_blank" rel="noopener noreferrer" class="block">
                              <img
                                src="https://i.ytimg.com/vi/{extractYouTubeId(video.link)}/hqdefault.jpg"
                                alt="Video preview"
                                class="w-32 h-20 object-cover rounded border border-white/10"
                                loading="lazy"
                              />
                            </a>
                          {:else}
                            <span class="text-white/40 text-xs">No preview</span>
                          {/if}
                        </td>
                        <td class="px-4 py-3 text-white/90 text-sm max-w-xs truncate">
                          <a href={video.link} target="_blank" rel="noopener noreferrer" class="hover:text-[#08407C] underline">
                            {video.link || '-'}
                          </a>
                        </td>
                        <td class="px-4 py-3 text-white/90 text-sm">{formatDate(video.expiration_at)}</td>
                        <td class="px-4 py-3 text-sm">
                          {#if video.active === 1}
                            <span class="px-2 py-1 rounded bg-green-500/20 text-green-300 text-xs font-semibold">Activo</span>
                          {:else}
                            <span class="px-2 py-1 rounded bg-gray-500/20 text-gray-400 text-xs font-semibold">Inactivo</span>
                          {/if}
                        </td>
                        <td class="px-4 py-3">
                          <div class="flex gap-2">
                            <button
                              on:click={() => openEditVideoForm(video)}
                              class="px-3 py-1 rounded bg-blue-500/20 hover:bg-blue-500/30 text-blue-300 text-xs font-semibold transition"
                            >
                              Editar
                            </button>
                            <button
                              on:click={() => openDeleteConfirm(video.id)}
                              class="px-3 py-1 rounded bg-red-500/20 hover:bg-red-500/30 text-red-300 text-xs font-semibold transition"
                            >
                              Borrar
                            </button>
                          </div>
                        </td>
                      </tr>
                    {/each}
                  </tbody>
                </table>
              </div>
            {/if}
          </div>
        </div>
      {/if}
    </div>
  {/if}
</div>

<!-- MODAL NEW VIDEO -->
{#if showNewVideoForm}
  <div class="fixed inset-0 z-50">
    <button class="absolute inset-0 bg-black/70" on:click={closeNewVideoForm} aria-label="Cerrar"></button>

    <div
      class="absolute left-1/2 top-1/2 w-[min(32rem,calc(100%-2rem))] -translate-x-1/2 -translate-y-1/2 rounded-2xl border border-white/10 bg-[#0b1220] p-6 text-white shadow-2xl"
      role="dialog"
      aria-modal="true"
    >
      <h4 class="text-xl font-bold">Crear Nuevo Video</h4>
      <p class="mt-2 text-white/70">Completa los datos del nuevo video.</p>

      <form on:submit={handleCreateVideo} class="mt-5 space-y-4">
        <label class="grid gap-2">
          <span class="text-sm text-white/70">Link del video</span>
          <input
            type="url"
            bind:value={newVideo.link}
            required
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
            placeholder="https://youtube.com/watch?v=..."
          />
        </label>

        <label class="grid gap-2">
          <span class="text-sm text-white/70">Fecha de expiración</span>
          <input
            type="datetime-local"
            bind:value={newVideo.expiration_at}
            required
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
          />
        </label>

        <label class="grid gap-2">
          <span class="text-sm text-white/70">Estado</span>
          <select
            bind:value={newVideo.active}
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
          >
            <option value={1}>Activo</option>
            <option value={0}>Inactivo</option>
          </select>
        </label>

        {#if createVideoError}
          <p class="text-red-400 text-sm">{createVideoError}</p>
        {/if}

        <div class="flex justify-end gap-3 pt-2">
          <button
            type="button"
            on:click={closeNewVideoForm}
            class="px-4 py-2 rounded-xl bg-white/10 hover:bg-white/15 border border-white/10 transition"
          >
            Cancelar
          </button>

          <button
            type="submit"
            disabled={isCreatingVideo}
            class="px-4 py-2 rounded-xl bg-[#08407C] hover:bg-[#0A4D95] text-white font-bold transition disabled:opacity-50"
          >
            {isCreatingVideo ? 'Creando...' : 'Crear Video'}
          </button>
        </div>
      </form>
    </div>
  </div>
{/if}

<!-- MODAL EDIT VIDEO -->
{#if showEditVideoForm && editingVideo}
  <div class="fixed inset-0 z-50">
    <button class="absolute inset-0 bg-black/70" on:click={closeEditVideoForm} aria-label="Cerrar"></button>

    <div
      class="absolute left-1/2 top-1/2 w-[min(32rem,calc(100%-2rem))] -translate-x-1/2 -translate-y-1/2 rounded-2xl border border-white/10 bg-[#0b1220] p-6 text-white shadow-2xl"
      role="dialog"
      aria-modal="true"
    >
      <h4 class="text-xl font-bold">Editar Video</h4>
      <p class="mt-2 text-white/70">Modifica los datos del video.</p>

      <form on:submit={handleEditVideo} class="mt-5 space-y-4">
        <label class="grid gap-2">
          <span class="text-sm text-white/70">Link del video</span>
          <input
            type="url"
            bind:value={editingVideo.link}
            required
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
            placeholder="https://youtube.com/watch?v=..."
          />
        </label>

        <label class="grid gap-2">
          <span class="text-sm text-white/70">Fecha de expiración</span>
          <input
            type="datetime-local"
            bind:value={editingVideo.expiration_at}
            required
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
          />
        </label>

        <label class="grid gap-2">
          <span class="text-sm text-white/70">Estado</span>
          <select
            bind:value={editingVideo.active}
            class="w-full px-4 py-3 rounded-xl border border-white/10 bg-white/5 text-white outline-none focus:border-white/30"
          >
            <option value={1}>Activo</option>
            <option value={0}>Inactivo</option>
          </select>
        </label>

        {#if editVideoError}
          <p class="text-red-400 text-sm">{editVideoError}</p>
        {/if}

        <div class="flex justify-end gap-3 pt-2">
          <button
            type="button"
            on:click={closeEditVideoForm}
            class="px-4 py-2 rounded-xl bg-white/10 hover:bg-white/15 border border-white/10 transition"
          >
            Cancelar
          </button>

          <button
            type="submit"
            disabled={isEditingVideo}
            class="px-4 py-2 rounded-xl bg-[#08407C] hover:bg-[#0A4D95] text-white font-bold transition disabled:opacity-50"
          >
            {isEditingVideo ? 'Guardando...' : 'Guardar Cambios'}
          </button>
        </div>
      </form>
    </div>
  </div>
{/if}

<!-- MODAL DELETE CONFIRM -->
{#if showDeleteConfirm}
  <div class="fixed inset-0 z-50">
    <button class="absolute inset-0 bg-black/70" on:click={closeDeleteConfirm} aria-label="Cerrar"></button>

    <div
      class="absolute left-1/2 top-1/2 w-[min(28rem,calc(100%-2rem))] -translate-x-1/2 -translate-y-1/2 rounded-2xl border border-white/10 bg-[#0b1220] p-6 text-white shadow-2xl"
      role="dialog"
      aria-modal="true"
    >
      <h4 class="text-xl font-bold text-red-400">¿Eliminar video?</h4>
      <p class="mt-3 text-white/70">Esta acción no se puede deshacer.</p>

      <div class="flex justify-end gap-3 mt-6">
        <button
          type="button"
          on:click={closeDeleteConfirm}
          disabled={isDeletingVideo}
          class="px-4 py-2 rounded-xl bg-white/10 hover:bg-white/15 border border-white/10 transition disabled:opacity-50"
        >
          Cancelar
        </button>

        <button
          type="button"
          on:click={handleDeleteVideo}
          disabled={isDeletingVideo}
          class="px-4 py-2 rounded-xl bg-red-600 hover:bg-red-700 text-white font-bold transition disabled:opacity-50"
        >
          {isDeletingVideo ? 'Eliminando...' : 'Eliminar'}
        </button>
      </div>
    </div>
  </div>
{/if}

<style>
  table {
    border-collapse: collapse;
  }
</style>
