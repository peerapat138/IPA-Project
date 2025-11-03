<svelte:head>
  <link rel="stylesheet" href="/style.css">
</svelte:head>
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    const API_BASE = "/api";

    setInterval(() => {
      const clock = document.getElementById('clock');
      if (clock) clock.textContent = new Date().toLocaleTimeString();
    }, 1000);

    const select = document.getElementById('traceroute-command');
    const paramInput = document.getElementById('traceroute-param-input');
    const traceBtn = document.getElementById('trace-btn');
    const consoleBox = document.getElementById('trace-console');
    const destInput = document.getElementById('destination-input');

    const optionsWithParam = ['-h','-j','-w','-S'];

    if (select) {
      select.addEventListener('change', (e) => {
        if (optionsWithParam.includes(e.target.value)) {
          paramInput.style.display = 'block';
          paramInput.placeholder = `Enter the value for ${e.target.value}`;
        } else {
          paramInput.style.display = 'none';
          paramInput.value = '';
        }
      });
    }

    if (traceBtn) {
      traceBtn.addEventListener('click', async () => {
        const destValue = destInput.value.trim();
        const command = select.value;
        const param = paramInput.value.trim();

        if (!destValue) {
          alert("⚠️ กรุณากรอก IP หรือ Hostname");
          return;
        }

        const targetArray = destValue
          .split(",")
          .map(s => s.trim())
          .filter(s => s.length > 0);

        let args = [];
        if (command) args = param ? [command, param] : [command];

        consoleBox.innerHTML = "⏳ Running traceroute test...";

        try {
          const res = await fetch(`${API_BASE}/trace`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ ip: targetArray, options: args })
          });

          const data = await res.json();

          if (res.ok && data.results && data.results.length > 0) {
            consoleBox.innerHTML = data.results.map((result, index) => `
              <div style="margin-bottom: 10px; border-bottom: 1px dashed #aaa; padding-bottom: 8px;">
                <b>#${index + 1}</b><br>
                <b>Target:</b> ${result.target || result.input}<br>
                <b>Status:</b> ${result.success ? "✅ Success" : "❌ Failed"}<br>
                <pre>${result.result}</pre>
              </div>
            `).join("");
          } else {
            consoleBox.innerHTML = `<span style="color:red">Error: ${data.error || "Unknown error"}</span>`;
          }
        } catch (err) {
          consoleBox.innerHTML = `<span style="color:red">Request failed: ${err.message}</span>`;
        }
      });
    }

    const openTableBtn = document.getElementById('open-nwtable');
    const popup = document.getElementById('nwtable-popup');
    const closeBtn = document.getElementById('nwtable-cancel-btn');
    const ipListContainer = document.getElementById('ip-list-container');

    if (openTableBtn) {
      openTableBtn.addEventListener('click', () => {
        populateIpList(ipListContainer, destInput);
        popup.classList.remove('hidden');
      });
    }

    if (closeBtn) {
      closeBtn.addEventListener('click', () => popup.classList.add('hidden'));
    }

    async function populateIpList(container, targetInput) {
      container.innerHTML = "<p>Loading...</p>";

      try {
        const res = await fetch(`${API_BASE}/networks`);
        const data = await res.json();

        if (!res.ok || !Array.isArray(data) || data.length === 0) {
          container.innerHTML = '<p>No entries found. Please add networks in the NETWORK-TABLE page.</p>';
          return;
        }

        container.innerHTML = '';

        data.forEach(net => {
          const wrapper = document.createElement('div');
          wrapper.className = 'ip-list-item';

          const title = document.createElement('div');
          title.textContent = `🌐 ${net.network_name}`;
          title.style.fontWeight = 'bold';
          title.style.marginBottom = '4px';
          wrapper.appendChild(title);

          if (Array.isArray(net.ip_list) && net.ip_list.length > 0) {
            const ipList = document.createElement('ul');
            ipList.style.margin = '0';
            ipList.style.paddingLeft = '20px';

            net.ip_list.forEach(ip => {
              const li = document.createElement('li');
              li.textContent = ip;
              ipList.appendChild(li);
            });
            wrapper.appendChild(ipList);

            wrapper.addEventListener('click', () => {
              const ipCombined = net.ip_list.map(ip => `${ip}`);
              targetInput.value = ipCombined.join(", ");
              popup.classList.add('hidden');
            });
          } else {
            const noIp = document.createElement('p');
            noIp.textContent = '(No IP entries)';
            noIp.style.color = '#777';
            wrapper.appendChild(noIp);
          }

          container.appendChild(wrapper);
        });
      } catch (err) {
        container.innerHTML = `<p style="color:red">Failed to load data: ${err.message}</p>`;
      }
    }
  });
</script>

<div class="content page">
    <div class="box">
        <h2 class="title">TRACEROUTE TEST</h2>
        <hr class="line">

        <div class="container">
            <div class="left">
                <h3>DESTINATION CONTROL</h3>
                <!-- svelte-ignore a11y_label_has_associated_control -->
                <label>IP Address / Hostname:</label>
                <input type="text" id="destination-input" placeholder="destination">
                <button class="small-btn" id="open-nwtable">USE NETWORK-TABLE</button>

                <label for="traceroute-command">Command:</label>
                <select id="traceroute-command">
                    <option value="">-</option>
                    <option value="-d">-d Do not resolve addresses to hostnames.</option>
                    <option value="-h">-h Maximum number of hops to search for target.</option>
                    <option value="-4">-4 Force using IPv4.</option>
                    <option value="-6">-6 Force using IPv6.</option>
                </select>

                <input type="text" id="traceroute-param-input" class="command-param-input"
                        placeholder="ใส่ค่า parameter (เช่น 20, host1, 1000)" style="display:none;">
                <button class="btn" id="trace-btn">TRACEROUTE</button>
            </div>

            <div class="right">
                <div class="console" id="trace-console">
                    Ready to trace route...
                </div>
            </div>
        </div>

        <div class="time">
            TIME: <span id="clock">12:45:51</span>
        </div>
    </div>
</div>

<div class="popup-nwtable hidden" id="nwtable-popup">
    <div class="popup-box">
        <h2>Select Destination from Network-Table</h2>
        <div class="ip-list-container" id="ip-list-container">
            <p>No entries found. Please add networks in the NETWORK-TABLE page.</p>
        </div>
        <button class="pop_clos" id="nwtable-cancel-btn">CLOSE</button>
    </div>
</div>
