<svelte:head>
  <link rel="stylesheet" href="/style.css">
</svelte:head>

<script>
  import { onMount } from 'svelte';

  const API_BASE = "/api"; // เปลี่ยนเป็น /api

  function createElementFromHTML(htmlString) {
    const div = document.createElement("div");
    div.innerHTML = htmlString.trim();
    return div.firstChild;
  }

  async function fetchJSON(url) {
    const res = await fetch(url);
    if (!res.ok) throw new Error("HTTP " + res.status);
    return await res.json();
  }

  onMount(() => {
    const popupNetwork = document.getElementById("popup-network");
    const popupIP = document.getElementById("popup-ip");
    const openPopupBtn = document.getElementById("open-popup");
    const cancelNetwork = document.getElementById("cancel-network");
    const cancelIP = document.getElementById("cancel-ip");
    const addNetworkForm = document.getElementById("add-network-form");
    const addIPForm = document.getElementById("add-ip-form");
    const container = document.getElementById("network-table-entries");
    const currentNetworkId = document.getElementById("current-network-id");

    openPopupBtn.addEventListener("click", () => popupNetwork.classList.remove("hidden"));
    cancelNetwork.addEventListener("click", (e) => {
      e.preventDefault();
      popupNetwork.classList.add("hidden");
      addNetworkForm.reset();
    });
    cancelIP.addEventListener("click", (e) => {
      e.preventDefault();
      popupIP.classList.add("hidden");
      addIPForm.reset();
    });
    window.addEventListener("click", (e) => {
      if (e.target === popupNetwork) popupNetwork.classList.add("hidden");
      if (e.target === popupIP) popupIP.classList.add("hidden");
    });

    async function toggleSubnet(btn) {
      const master = btn.closest(".network-item");
      const networkId = master.dataset.networkId;
      const isCollapsed = btn.textContent === "∨";

      if (isCollapsed) {
        const ips = await fetchJSON(`${API_BASE}/ips/${networkId}`);
        document.querySelectorAll(`.network-item.detail[data-network-id="${networkId}"]`).forEach(d => d.remove());

        for (const ip of ips) {
          const ipHTML = `
            <div class="network-item detail" data-network-id="${networkId}">
              <div class="network-header">
                <span>${ip.ip_address}</span>
                <button class="remove">—</button>
              </div>
            </div>`;
          const ipEl = createElementFromHTML(ipHTML);
          master.insertAdjacentElement("afterend", ipEl);
          attachRemoveListener(ipEl.querySelector(".remove"), ip.entry_id, networkId);
        }
      } else {
        document.querySelectorAll(`.network-item.detail[data-network-id="${networkId}"]`).forEach(d => d.remove());
      }
      btn.textContent = isCollapsed ? "∧" : "∨";
    }

    function attachToggleListener(btn) {
      btn.addEventListener("click", () => toggleSubnet(btn));
    }

    function attachRemoveListener(btn, id, networkId) {
      btn.addEventListener("click", async (event) => {
        event.preventDefault();

        try {
          const res = await fetch(`${API_BASE}/ips/${id}`, { method: "DELETE" });
          if (!res.ok) throw new Error("Failed to delete IP");

          btn.closest(".network-item.detail").remove();

          const remain = document.querySelectorAll(`.network-item.detail[data-network-id="${networkId}"]`);
          if (remain.length === 0) {
            document.querySelector(`.network-item[data-network-id="${networkId}"]:not(.detail)`)?.remove();
          }

          alert("✅ ลบ IP สำเร็จ");
        } catch (err) {
          console.error(err);
          alert("❌ ลบ IP ไม่สำเร็จ");
        }
      });
    }

    function attachAddIPListener(btn, networkId) {
      btn.addEventListener("click", () => {
        popupIP.classList.remove("hidden");
        currentNetworkId.value = networkId;
      });
    }

    function attachRemoveNetworkListener(btn, networkId) {
      btn.addEventListener("click", async () => {
        if (!confirm("ต้องการลบ Network นี้จริงหรือไม่?")) return;

        const res = await fetch(`${API_BASE}/networks/${networkId}`, { method: "DELETE" });
        if (res.ok) {
          document.querySelector(`.network-item[data-network-id="${networkId}"]:not(.detail)`)?.remove();
          document.querySelectorAll(`.network-item.detail[data-network-id="${networkId}"]`).forEach(e => e.remove());
        } else {
          alert("Error deleting network");
        }
      });
    }

    async function loadNetworks() {
      container.querySelectorAll(".network-item").forEach(e => e.remove());
      const networks = await fetchJSON(`${API_BASE}/networks`);

      for (const net of networks) {
        const masterHTML = `
          <div class="network-item" data-network-id="${net.network_id}">
            <div class="network-header">
              <span>${net.network_name}</span>
              <div class="actions">
                <button class="toggle">∨</button>
                <button class="add-ip">+</button>
                <button class="remove-network">—</button>
              </div>
            </div>
          </div>`;
        const master = createElementFromHTML(masterHTML);
        container.insertBefore(master, openPopupBtn);

        attachToggleListener(master.querySelector(".toggle"));
        attachAddIPListener(master.querySelector(".add-ip"), net.network_id);
        attachRemoveNetworkListener(master.querySelector(".remove-network"), net.network_id);
      }
    }

    addNetworkForm.addEventListener("submit", async (e) => {
      e.preventDefault();
      const name = document.getElementById("network-input").value.trim();
      if (!name) return;
      const res = await fetch(`${API_BASE}/networks`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ network_name: name })
      });
      if (res.ok) {
        popupNetwork.classList.add("hidden");
        addNetworkForm.reset();
        await loadNetworks();
      } else {
        const err = await res.json();
        alert(err.error || "Error adding network");
      }
    });

    addIPForm.addEventListener("submit", async (e) => {
      e.preventDefault();
      const ip = document.getElementById("ip-input").value.trim();
      const networkId = currentNetworkId.value;
      if (!ip) return;

      try {
        const res = await fetch(`${API_BASE}/ips`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ network_id: networkId, ip_address: ip })
        });

        const data = await res.json();

        if (res.ok) {
          alert(`✅ เพิ่ม IP ${ip} สำเร็จ`);
          popupIP.classList.add("hidden");
          addIPForm.reset();

          const toggleBtn = document.querySelector(`.network-item[data-network-id="${networkId}"] .toggle`);
          if (toggleBtn && toggleBtn.textContent === "∧") {
            await toggleSubnet(toggleBtn);
            toggleBtn.textContent = "∧";
          }
        } else {
          alert(`❌ เพิ่ม IP ไม่สำเร็จ: ${data.error || "Unknown error"}`);
        }
      } catch (err) {
        alert(`⚠️ Request failed: ${err.message}`);
      }
    });

    loadNetworks();
  });
</script>

<div class="content page">
  <div class="box">
    <h2 class="title">NETWORK-TABLE</h2>
    <hr class="line" />

    <main class="content">
      <div class="network-container" id="network-table-entries">
        <button id="open-popup" class="open-popup">+ Add Network Table</button>
      </div>
    </main>
  </div>
</div>

<!-- Popups -->
<div class="popup hidden" id="popup-network">
  <div class="popup-box">
    <h2>Add Network Table</h2>
    <form id="add-network-form">
      <!-- svelte-ignore a11y_label_has_associated_control -->
      <label>Network Name:</label>
      <input type="text" id="network-input" placeholder="e.g. 10.30.6.0/23" required />
      <button class="add">ADD</button>
      <button class="close" id="cancel-network">CANCEL</button>
    </form>
  </div>
</div>

<div class="popup hidden" id="popup-ip">
  <div class="popup-box">
    <h2>Add IP Address / Domain Name</h2>
    <form id="add-ip-form">
      <input type="hidden" id="current-network-id" />
      <!-- svelte-ignore a11y_label_has_associated_control -->
      <label>IP Address / Domain Name:</label>
      <input type="text" id="ip-input" placeholder="e.g. 10.30.6.10 / www.google.com" required />
      <button class="add">ADD</button>
      <button class="close" id="cancel-ip">CANCEL</button>
    </form>
  </div>
</div>