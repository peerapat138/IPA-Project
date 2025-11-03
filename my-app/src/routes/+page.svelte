<script>
    import { page } from '$app/stores';
</script>

<svelte:head>
  <link rel="stylesheet" href="/style.css">
</svelte:head>

  <div class="content">
    <div class="box">
      <div class="logo">
        <div class="bolt">
          <svg width="38" height="56" viewBox="0 0 24 36" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M12 0L0 18h9l-1 18L24 12h-9l-3.5-10z" fill="#22FFF0"/>
          </svg>
        </div>
        <div>
          <h1>Network Tester</h1>
          <p class="subtitle">Let us help you!</p>
        </div>
      </div>

      <div class="actions">
        <a class="neon-btn" href="/ping">PING</a>
        <a class="neon-btn" href="/traceroute">TRACEROUTE</a>
        <a class="neon-btn" href="/dns">DNS</a>
        <a class="neon-btn" href="/nwtable">NETWORK-TABLE</a>
      </div>
    </div>
  </div>
