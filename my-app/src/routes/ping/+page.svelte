<svelte:head>
  <link rel="stylesheet" href="/style.css">
</svelte:head>

<script>
    const API_BASE = "/api"; // เปลี่ยนเป็น /api

    import { onMount } from 'svelte';
    onMount(() => {
        setInterval(() => {
        const clockEl = document.getElementById("clock");
        if (clockEl) clockEl.textContent = new Date().toLocaleTimeString();
        }, 1000);

    const select = document.getElementById("ping-command");
    const paramInput = document.getElementById("ping-param-input");
    const openTableBtn = document.querySelector(".small-btn");
    const destinationInput = document.querySelector(".left input[type='text']");
    const popup = document.getElementById("nwtable-popup");
    const closeBtn = document.getElementById("nwtable-cancel-btn");
    const ipListContainer = document.getElementById("ip-list-container");
    const pingBtn = document.querySelector(".btn");
    const consoleBox = document.querySelector(".console");

    const optionsWithParam = ["-n", "-l", "-i", "-w"];

    select?.addEventListener("change", (e) => {
      const selectedValue = e.target.value;
      if (optionsWithParam.includes(selectedValue)) {
        paramInput.style.display = "block";
        paramInput.placeholder = `Enter the value for ${selectedValue}`;
      } else {
        paramInput.style.display = "none";
        paramInput.value = "";
      }
    });

    openTableBtn?.addEventListener("click", async () => {
      await populateIpList(ipListContainer, destinationInput);
      popup.classList.remove("hidden");
    });

    closeBtn?.addEventListener("click", () => popup.classList.add("hidden"));

    async function populateIpList(container, targetInput) {
      container.innerHTML = "<p>Loading...</p>";

      try {
        const res = await fetch(`${API_BASE}/networks`);
        const data = await res.json();

        if (!res.ok || !Array.isArray(data) || data.length === 0) {
          container.innerHTML =
            "<p>No entries found. Please add networks in the NETWORK-TABLE page.</p>";
          return;
        }

        container.innerHTML = "";

        data.forEach((net) => {
          const wrapper = document.createElement("div");
          wrapper.className = "ip-list-item";

          const title = document.createElement("div");
          title.textContent = `🌐 ${net.network_name}`;
          title.style.fontWeight = "bold";
          title.style.marginBottom = "4px";
          wrapper.appendChild(title);

          if (Array.isArray(net.ip_list) && net.ip_list.length > 0) {
            const ipList = document.createElement("ul");
            ipList.style.margin = "0";
            ipList.style.paddingLeft = "20px";

            net.ip_list.forEach((ip) => {
              const li = document.createElement("li");
              li.textContent = ip;
              ipList.appendChild(li);
            });
            wrapper.appendChild(ipList);

            wrapper.addEventListener("click", () => {
              const ipCombined = net.ip_list.map((ip) => `${ip}`);
              targetInput.value = ipCombined.join(", ");
              popup.classList.add("hidden");
            });
          } else {
            const noIp = document.createElement("p");
            noIp.textContent = "(No IP entries)";
            noIp.style.color = "#777";
            wrapper.appendChild(noIp);
          }

          container.appendChild(wrapper);
        });
      } catch (err) {
        container.innerHTML = `<p style="color:red">Failed to load data: ${err.message}</p>`;
      }
    }

    pingBtn?.addEventListener("click", async () => {
      const ipValue = destinationInput.value.trim();
      const command = select.value;
      const param = paramInput.value.trim();

      if (!ipValue) {
        alert("⚠️ กรุณากรอก IP Address ก่อน");
        return;
      }

      let args = [];
      if (command && param) args = [command, param];
      const ipList = ipValue
        .split(",")
        .map((ip) => ip.trim())
        .filter((ip) => ip.length > 0);

      consoleBox.innerHTML = "⏳ Running ping test...";

      try {
        const res = await fetch(`${API_BASE}/ping`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ targets: ipList, rawArgs: args }),
        });

        const data = await res.json();

        if (res.ok && data.results && data.results.length > 0) {
          consoleBox.innerHTML = data.results
            .map(
              (result, index) => `
              <div style="margin-bottom: 10px; border-bottom: 1px dashed #aaa; padding-bottom: 8px;">
                <b>#${index + 1}</b><br>
                <b>Target:</b> ${result.target || result.input}<br>
                <b>Mask:</b> ${result.mask || "-"}<br>
                <b>Status:</b> ${
                  result.success ? "✅ Success" : "❌ Failed"
                }<br>
                <pre>${result.result}</pre>
              </div>`
            )
            .join("");
        } else {
          consoleBox.innerHTML = `<span style="color:red">Error: ${
            data.error || "Unknown error"
          }</span>`;
        }
      } catch (err) {
        consoleBox.innerHTML = `<span style="color:red">Request failed: ${err.message}</span>`;
      }
    });
  });
</script>
<div class="content page">
    <div class="box">
        <h2 class="title">PING TEST</h2>
        <hr class="line">

        <div class="container">
            <div class="left">
                <h3>DESTINATION CONTROL</h3>
                <!-- svelte-ignore a11y_label_has_associated_control -->
                <label>IP Address:</label>
                <input type="text" placeholder="destination">
                <button class="small-btn">USE NETWORK-TABLE</button>

                <label for="ping-command">Command:</label>
                <select id="ping-command">
                    <option value="">-</option>
                    <option value="-n">-n count (Number of echo requests)</option>
                    <option value="-l">-l size (Send buffer size)</option>
                    <option value="-i">-i TTL (Time To Live)</option>
                    <option value="-w">-w timeout (milliseconds)</option>                          
                </select>

                <input type="text" id="ping-param-input" class="command-param-input"
                    placeholder="ใส่ค่า parameter (เช่น 10, 1000, 64)">

                <button class="btn">PING</button>
            </div>

            <div class="right">
                <div class="console">
                    Output Console:<br>
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