<script>
  import { onDestroy } from 'svelte';

  // Countdown hacia el 12 de febrero de 2026
  let days = '00';
  let hours = '00';
  let minutes = '00';
  let seconds = '00';

  const countDownDate = new Date('Feb 12, 2026 00:00:00').getTime();

  const countdownFunction = setInterval(() => {
    const now = new Date().getTime();
    const distance = countDownDate - now;

    if (distance < 0) {
      clearInterval(countdownFunction);
      days = hours = minutes = seconds = '00';
      return;
    }

    days = Math.floor(distance / (1000 * 60 * 60 * 24))
      .toString()
      .padStart(2, '0');
    hours = Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60))
      .toString()
      .padStart(2, '0');
    minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60))
      .toString()
      .padStart(2, '0');
    seconds = Math.floor((distance % (1000 * 60)) / 1000)
      .toString()
      .padStart(2, '0');
  }, 1000);

  onDestroy(() => clearInterval(countdownFunction));

  // ==========================
  // Limite de palabras (Comentario)
  // ==========================
  const MAX_WORDS = 80;
  const MAX_CHARS = 600; // backup por seguridad

  let comentario = '';
  let comentarioWords = 0;

  function countWords(text) {
    return text.trim().split(/\s+/).filter(Boolean).length;
  }

  function clampWords(text, maxWords) {
    const words = text.trim().split(/\s+/).filter(Boolean);
    if (words.length <= maxWords) return text;
    return words.slice(0, maxWords).join(' ');
  }

  function handleComentarioInput(e) {
    let value = e.currentTarget.value;

    // backup por caracteres
    if (value.length > MAX_CHARS) value = value.slice(0, MAX_CHARS);

    // límite real por palabras
    value = clampWords(value, MAX_WORDS);

    comentario = value;
    comentarioWords = countWords(comentario);
  }

  // ==========================
  // Anillo de seguridad (Form)
  // ==========================
  let isSubmitting = false;
  let submitError = '';
  let submitSuccess = false;

  const WEBHOOK_URL = 'https://n8n-krak.com/webhook/workshop-newsletter';
  const FETCH_TIMEOUT_MS = 15000;

  // Manejo del formulario
  async function handleForm(e) {
    e.preventDefault();

    // Anti doble-submit (usuarios ansiosos / doble click / lag)
    if (isSubmitting) return;
    isSubmitting = true;

    // Reset mensajes UI
    submitError = '';
    submitSuccess = false;

    const formEl = e.currentTarget;
    const form = new FormData(formEl);

    // opcional: agregamos metadata útil
    form.append('source', 'agentes.krak.com.ar');
    form.append('ts', new Date().toISOString());

    // (extra safety) por si el browser mandara otro valor
    // garantizamos que salga el comentario ya limitado
    form.set('comentario', comentario);

    // Timeout para evitar requests colgados (reduce reintentos y duplicados)
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);

    try {
      const res = await fetch(WEBHOOK_URL, {
        method: 'POST',
        body: form, // <- FormData, SIN headers
        signal: controller.signal,
      });

      clearTimeout(timeout);

      if (!res.ok) {
        const msg = await res.text().catch(() => '');
        console.error('Webhook error:', res.status, msg);

        submitError = 'Hubo un error. Por favor, intentá nuevamente.';
        alert(submitError);
        return;
      }

      // Forward-compatible: si n8n en el futuro devuelve JSON { ok: true }
      // (si no hay JSON, no pasa nada)
      const data = await res.clone().json().catch(() => null);
      if (data && data.ok === false) {
        console.error('Webhook response not ok:', data);

        submitError = 'Hubo un error. Por favor, intentá nuevamente.';
        alert(submitError);
        return;
      }

      // ✅ Meta: conversión real (Lead) SOLO cuando el webhook respondió OK.
      // Nota team: el Pixel base se carga globalmente en src/app.html.
      // Importante: NO medir conversiones por clicks en CTAs (hero/beneficios).
      if (typeof window !== 'undefined' && window.fbq) {
        window.fbq('track', 'Lead');
        // Alternativa semántica si MKT la prefiere:
        // window.fbq('track', 'CompleteRegistration');
      }

      submitSuccess = true;

      alert('Gracias por inscribirte!');
      formEl.reset();
      comentario = '';
      comentarioWords = 0;
    } catch (error) {
      clearTimeout(timeout);
      console.error(error);

      submitError =
        error?.name === 'AbortError'
          ? 'La respuesta tardó demasiado. Probá nuevamente en unos segundos.'
          : 'Hubo un error. Por favor, intentá nuevamente.';

      alert(submitError);
    } finally {
      isSubmitting = false;
    }
  }
</script>

<svelte:head>
  <title>Masterclass de Krak Real Estate con Marcelo Napolitano</title>
  <meta
    name="description"
    content="Masterclass para agentes inmobiliarios: estrategias para generar confianza, manejar objeciones y acelerar cierres con seguridad y autoridad."
  />
</svelte:head>

<!-- HERO (dark + glow Krak) -->
<section class="relative overflow-hidden bg-[#050A14] py-20 sm:py-24 px-4 sm:px-6 lg:px-8">
  <!-- glows -->
  <div
    class="pointer-events-none absolute -top-44 left-1/2 h-[520px] w-[520px] -translate-x-1/2 rounded-full bg-[#08407C]/30 blur-[120px]"
  ></div>
  <div
    class="pointer-events-none absolute -bottom-60 right-[-120px] h-[520px] w-[520px] rounded-full bg-[#08407C]/25 blur-[140px]"
  ></div>

  <div class="relative mx-auto max-w-5xl text-center">
    <p
      class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-white/5 px-4 py-2 text-sm text-white/80"
    >
      Masterclass para agentes inmobiliarios
      <span class="h-1 w-1 rounded-full bg-white/40"></span>
      Mentor: Marcelo Napolitano
    </p>

    <h1 class="mt-7 text-4xl sm:text-5xl md:text-6xl font-bold text-white font-inter leading-tight">
      Convertite en el agente inmobiliario que cierra con seguridad y autoridad
    </h1>

    <p class="mt-6 text-lg sm:text-xl md:text-2xl text-white/75 font-inter max-w-3xl mx-auto">
      Hoy tenés clientes, propiedades y oportunidades
      <br class="hidden md:block" />
      pero cerrar operaciones sigue siendo más difícil de lo que debería.
    </p>

    <div class="mt-10 flex flex-col sm:flex-row items-center justify-center gap-4">
      <a
        href="#formulario"
        class="inline-flex items-center justify-center rounded-xl bg-[#08407C] hover:bg-[#0A4D95] text-white font-bold text-lg py-4 px-8 shadow-lg shadow-[#08407C]/30 transition duration-300 transform hover:scale-[1.02] border border-white/10 w-full sm:w-auto"
      >
        Inscribite gratis y empezá a jugar en serio
      </a>

      <a
        href="#video"
        class="inline-flex items-center justify-center rounded-xl bg-white/5 hover:bg-white/10 text-white font-semibold text-lg py-4 px-8 transition duration-300 border border-white/10 w-full sm:w-auto"
      >
        Ver el video
      </a>
    </div>

    <p class="mt-6 text-sm text-white/50">
      Cupos limitados. Reservá tu lugar y entrá con claridad al próximo cierre.
    </p>
  </div>
</section>

<!-- VALOR / MENTOR -->
<section class="py-14 sm:py-16 px-4 sm:px-6 lg:px-8 bg-[#070F1F] text-white">
  <div class="max-w-5xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-10 items-center">
    <div>
      <h2 class="text-3xl md:text-4xl font-bold font-inter leading-tight">
        Aprendé a cerrar sin dudar, sin improvisar y sin desgastarte
      </h2>

      <p class="mt-5 text-lg text-white/75 font-inter leading-relaxed">
        En esta masterclass vas a aprender las estrategias reales que usan los agentes que inspiran confianza,
        negocian sin miedo y aceleran cierres sin fricción.
      </p>

      <div class="mt-7 rounded-2xl border border-white/10 bg-white/5 p-6">
        <p class="text-sm uppercase tracking-wider text-white/60 font-semibold">Mentor</p>
        <p class="mt-2 text-lg text-white/85 font-inter">
          Marcelo Napolitano te va a mostrar cómo pensar, cómo hablar y cómo actuar como un profesional
          que domina su mercado.
        </p>
      </div>
    </div>

    <!-- Fotos Marcelo (ya linkeadas) -->
    <div class="relative">
      <div class="absolute -inset-6 rounded-3xl bg-[#08407C]/20 blur-2xl"></div>

      <div class="relative rounded-3xl border border-white/10 bg-white/5 p-4">
        <!-- En mobile se apilan (1 col), desde sm pasan a 2 col -->
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <div class="aspect-[4/5] rounded-2xl bg-white/10 border border-white/10 overflow-hidden">
            <img
              src="/marcelo-hero.jpg"
              alt="Marcelo Napolitano"
              class="h-full w-full object-cover"
              loading="lazy"
            />
          </div>

          <div class="aspect-[4/5] rounded-2xl bg-white/10 border border-white/10 overflow-hidden">
            <img
              src="/marcelo-mentor.jpg"
              alt="Marcelo Napolitano"
              class="h-full w-full object-cover"
              loading="lazy"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- COUNTDOWN (lindo, prolijo, serio) -->
<section class="py-16 md:py-20 px-4 sm:px-6 lg:px-8 bg-[#050A14]">
  <div class="max-w-5xl mx-auto">
    <div class="rounded-3xl border border-white/10 bg-white/5 p-8 md:p-10 text-center relative overflow-hidden">
      <div
        class="pointer-events-none absolute -top-24 left-1/2 h-[320px] w-[320px] -translate-x-1/2 rounded-full bg-[#08407C]/20 blur-[90px]"
      ></div>

      <h2 class="relative text-3xl md:text-4xl font-bold text-white font-inter">⏰ Falta poco para dar el salto</h2>

      <div class="relative mt-10 grid grid-cols-2 md:grid-cols-4 gap-6">
        <div class="rounded-2xl border border-white/10 bg-black/20 p-6">
          <div class="text-6xl md:text-7xl font-bold text-white">{days}</div>
          <div class="mt-2 text-sm text-white/60">días</div>
        </div>

        <div class="rounded-2xl border border-white/10 bg-black/20 p-6">
          <div class="text-6xl md:text-7xl font-bold text-white">{hours}</div>
          <div class="mt-2 text-sm text-white/60">horas</div>
        </div>

        <div class="rounded-2xl border border-white/10 bg-black/20 p-6">
          <div class="text-6xl md:text-7xl font-bold text-white">{minutes}</div>
          <div class="mt-2 text-sm text-white/60">minutos</div>
        </div>

        <div class="rounded-2xl border border-white/10 bg-black/20 p-6">
          <div class="text-6xl md:text-7xl font-bold text-white">{seconds}</div>
          <div class="mt-2 text-sm text-white/60">segundos</div>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- VIDEO (debajo del contador) - SIN TÍTULO -->
<section id="video" class="py-14 sm:py-16 px-4 sm:px-6 lg:px-8 bg-[#070F1F]">
  <div class="max-w-4xl mx-auto">
    <div class="relative rounded-2xl overflow-hidden border border-white/10 bg-black">
      <div class="pointer-events-none absolute -inset-8 bg-[#08407C]/15 blur-2xl"></div>

      <!-- Aspect ratio fijo (responsive) -->
      <div class="relative aspect-video">
        <video class="absolute inset-0 w-full h-full object-cover" controls preload="metadata" muted playsinline>
          <source src="Masterclasslanding.mp4" type="video/mp4" />
          Tu navegador no soporta el elemento de video.
        </video>
      </div>
    </div>
  </div>
</section>

<!-- BENEFICIOS -->
<section class="py-14 sm:py-16 px-4 sm:px-6 lg:px-8 bg-[#050A14]">
  <div class="max-w-5xl mx-auto">
    <h2 class="text-3xl md:text-4xl font-bold text-white text-center font-inter">
      Lo que cambia después de esta masterclass
    </h2>

    <div class="mt-10 grid grid-cols-1 md:grid-cols-2 gap-6">
      <div class="rounded-2xl border border-white/10 bg-white/5 p-6">
        <p class="text-white font-semibold text-lg">Generar confianza desde el primer contacto</p>
        <p class="mt-2 text-white/70">
          Entrar a cada charla con una forma clara de presentarte, encuadrar la conversación y marcar el ritmo.
        </p>
      </div>

      <div class="rounded-2xl border border-white/10 bg-white/5 p-6">
        <p class="text-white font-semibold text-lg">Manejar objeciones sin improvisar</p>
        <p class="mt-2 text-white/70">
          Responder con argumentos sólidos, sin ponerte a la defensiva ni regalar autoridad.
        </p>
      </div>

      <div class="rounded-2xl border border-white/10 bg-white/5 p-6">
        <p class="text-white font-semibold text-lg">Cerrar propiedades más rápido y con menos fricción</p>
        <p class="mt-2 text-white/70">
          Reducir vueltas, evitar desgaste y avanzar hacia el cierre con decisiones más claras.
        </p>
      </div>

      <div class="rounded-2xl border border-white/10 bg-white/5 p-6">
        <p class="text-white font-semibold text-lg">Posicionarte como referente, no como “uno más”</p>
        <p class="mt-2 text-white/70">
          Elevar tu estándar: cómo hablás, cómo proponés y cómo te plantás en el mercado.
        </p>
      </div>
    </div>

    <div class="mt-10 text-center">
      <a
        href="#formulario"
        class="inline-flex items-center justify-center rounded-xl bg-[#08407C] hover:bg-[#0A4D95] text-white font-bold text-lg py-4 px-8 shadow-lg shadow-[#08407C]/30 transition duration-300 transform hover:scale-[1.02] border border-white/10"
      >
        Inscribite gratis y empezá a jugar en serio
      </a>
    </div>
  </div>
</section>

<!-- FORMULARIO -->
<section id="formulario" class="py-16 sm:py-20 px-4 sm:px-6 lg:px-8 bg-[#f0f4f8]">
  <div class="max-w-3xl mx-auto">
    <h2 class="text-3xl sm:text-4xl font-bold mb-10 text-center font-inter text-gray-900">
      Completá el formulario para reservar tu lugar
    </h2>

    <form on:submit={handleForm} class="space-y-8 bg-white p-6 sm:p-8 rounded-xl shadow-md">
      {#if submitError}
        <div class="rounded-lg border border-red-200 bg-red-50 text-red-700 px-4 py-3 text-sm">
          {submitError}
        </div>
      {/if}

      {#if submitSuccess}
        <div class="rounded-lg border border-green-200 bg-green-50 text-green-800 px-4 py-3 text-sm">
          ¡Listo! Te registramos correctamente.
        </div>
      {/if}

      <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
        <div>
          <label for="nombre" class="block text-lg font-medium text-gray-700">Nombre</label>
          <input
            type="text"
            id="nombre"
            name="nombre"
            required
            class="mt-2 block w-full border-2 border-gray-300 rounded-lg shadow-sm py-3 px-4 focus:outline-none focus:ring-2 focus:ring-[#08407C] focus:border-[#08407C] text-lg"
          />
        </div>

        <div>
          <label for="apellido" class="block text-lg font-medium text-gray-700">Apellido</label>
          <input
            type="text"
            id="apellido"
            name="apellido"
            required
            class="mt-2 block w-full border-2 border-gray-300 rounded-lg shadow-sm py-3 px-4 focus:outline-none focus:ring-2 focus:ring-[#08407C] focus:border-[#08407C] text-lg"
          />
        </div>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
        <div>
          <label for="email" class="block text-lg font-medium text-gray-700">Email</label>
          <input
            type="email"
            id="email"
            name="email"
            required
            class="mt-2 block w-full border-2 border-gray-300 rounded-lg shadow-sm py-3 px-4 focus:outline-none focus:ring-2 focus:ring-[#08407C] focus:border-[#08407C] text-lg"
          />
        </div>

        <div>
          <label for="telefono" class="block text-lg font-medium text-gray-700">Teléfono</label>
          <input
            type="tel"
            id="telefono"
            name="telefono"
            required
            class="mt-2 block w-full border-2 border-gray-300 rounded-lg shadow-sm py-3 px-4 focus:outline-none focus:ring-2 focus:ring-[#08407C] focus:border-[#08407C] text-lg"
          />
        </div>
      </div>

      <div>
        <label for="comentario" class="block text-lg font-medium text-gray-700">Comentario</label>
        <textarea
          id="comentario"
          name="comentario"
          rows="4"
          bind:value={comentario}
          on:input={handleComentarioInput}
          maxlength={MAX_CHARS}
          class="mt-2 block w-full border-2 border-gray-300 rounded-lg shadow-sm py-3 px-4 focus:outline-none focus:ring-2 focus:ring-[#08407C] focus:border-[#08407C] text-lg"
        ></textarea>

        <p class="mt-2 text-sm text-gray-500">
          {comentarioWords}/{MAX_WORDS} palabras
        </p>
      </div>

      <div>
        <button
          type="submit"
          disabled={isSubmitting}
          class="w-full bg-white hover:bg-gray-100 text-[#08407C] font-bold text-xl py-4 px-6 rounded-lg shadow-lg transition duration-300 transform hover:scale-[1.02] border-2 border-[#08407C] disabled:opacity-60 disabled:cursor-not-allowed"
        >
          {isSubmitting ? 'Enviando...' : 'Inscribite gratis y empezá a jugar en serio'}
        </button>
      </div>
    </form>
  </div>
</section>