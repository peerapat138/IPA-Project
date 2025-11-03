<script>
    import { page } from '$app/stores';
</script>

<div class="frame">
	<nav>
	<a href="/" class={$page.url.pathname === '/' ? 'active' : ''}>HOME</a>
	<a href="/ping" class={$page.url.pathname === '/ping' ? 'active' : ''}>PING</a>
	<a href="/traceroute" class={$page.url.pathname === '/traceroute' ? 'active' : ''}>TRACEROUTE</a>
	<a href="/dns" class={$page.url.pathname === '/dns' ? 'active' : ''}>DNS</a>
	<a href="/nwtable" class={$page.url.pathname === '/nwtable' ? 'active' : ''}>NETWORK-TABLE</a>
	</nav>
	<slot />
</div>

