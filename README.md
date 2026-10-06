# GHJ
Booss
// ==UserScript==
// @name         Email Scraper by Osayuki
// @version      7.0
// @description  Professional moveable/resizable lead-gen tool with tabbed navigation
// @author       Osayuki
// @match        *://*/*
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_addStyle
// @grant        GM_openInTab
// ==/UserScript==

(function() {
    'use strict';

    // --- CONFIG & STATE ---
    let leads = JSON.parse(GM_getValue("leads", "[]"));
    let config = JSON.parse(GM_getValue("scraper_config", '{"emails":true, "phones":false, "context":false, "opacity":0.95, "pos":{"t":"50px","l":"50px"}, "size":{"w":"350px","h":"450px"}}'));

    GM_addStyle(`
        #osayuki-root { position: fixed; top: ${config.pos.t}; left: ${config.pos.l}; width: ${config.size.w}; height: ${config.size.h}; 
                        z-index: 2147483647; background: rgba(15, 23, 42, ${config.opacity}); color: #f8fafc; 
                        border: 1px solid #334155; border-radius: 12px; display: flex; flex-direction: column; 
                        box-shadow: 0 20px 50px rgba(0,0,0,0.6); overflow: hidden; font-family: 'Segoe UI', Roboto, sans-serif; transition: opacity 0.3s; }
        #osayuki-header { background: #020617; padding: 12px; cursor: move; display: flex; justify-content: space-between; align-items: center; user-select: none; border-bottom: 1px solid #1e293b; }
        #osayuki-nav { display: flex; background: #0f172a; border-bottom: 1px solid #1e293b; }
        .nav-tab { flex: 1; padding: 10px 5px; text-align: center; cursor: pointer; font-size: 11px; font-weight: 600; text-transform: uppercase; color: #94a3b8; transition: 0.2s; }
        .nav-tab.active { color: #3b82f6; border-bottom: 2px solid #3b82f6; background: #1e293b; }
        #osayuki-body { flex: 1; overflow-y: auto; padding: 15px; scrollbar-width: thin; }
        .lead-card { background: #1e293b; padding: 10px; border-radius: 6px; margin-bottom: 10px; border: 1px solid #334155; position: relative; }
        .copy-btn { position: absolute; right: 5px; top: 5px; font-size: 10px; background: #334155; border: none; color: white; border-radius: 3px; cursor: pointer; }
        .action-row { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-bottom: 15px; }
        .btn { padding: 8px; border-radius: 6px; border: none; font-weight: bold; cursor: pointer; font-size: 12px; }
        .btn-blue { background: #2563eb; color: white; }
        .btn-red { background: #dc2626; color: white; }
        .btn-gray { background: #475569; color: white; }
        #osayuki-resize { width: 20px; height: 20px; position: absolute; bottom: 0; right: 0; cursor: nwse-resize; background: linear-gradient(135deg, transparent 50%, #475569 50%); }
        .hidden { display: none !important; }
    `);

    const root = document.createElement('div');
    root.id = 'osayuki-root';
    root.innerHTML = `
        <div id="osayuki-header">
            <span style="font-size: 13px; letter-spacing: 0.5px;">📧 EMAIL SCRAPER <span style="color:#3b82f6">BY OSAYUKI</span></span>
            <div style="display:flex; gap:12px; font-weight:bold;">
                <span id="os-min" style="cursor:pointer; color:#94a3b8">_</span>
                <span id="os-close" style="cursor:pointer; color:#ef4444">✕</span>
            </div>
        </div>
        <div id="osayuki-nav">
            <div class="nav-tab active" data-page="dash">Stats</div>
            <div class="nav-tab" data-page="emails">Emails</div>
            <div class="nav-tab" data-page="phones">Phones</div>
            <div class="nav-tab" data-page="settings">Settings</div>
        </div>
        <div id="osayuki-body">
            <div id="p-dash" class="tab-page">
                <div style="text-align:center; margin-bottom:20px;">
                    <div style="font-size:32px; font-weight:bold;" id="count-total">0</div>
                    <div style="color:#94a3b8; font-size:12px;">TOTAL LEADS COLLECTED</div>
                </div>
                <div class="action-row">
                    <button class="btn btn-blue" id="btn-copy-all">Copy All Emails</button>
                    <button class="btn btn-blue" id="btn-dl-csv">Export CSV</button>
                </div>
            </div>
            <div id="p-emails" class="tab-page hidden"></div>
            <div id="p-phones" class="tab-page hidden"></div>
            <div id="p-settings" class="tab-page hidden">
                <p style="font-size:11px; color:#94a3b8; margin-bottom:10px;">SCRAPING TARGETS</p>
                <label style="display:block; margin-bottom:8px;"><input type="checkbox" id="check-e" ${config.emails?'checked':''}> Emails (Gmail/Outlook)</label>
                <label style="display:block; margin-bottom:15px;"><input type="checkbox" id="check-p" ${config.phones?'checked':''}> Phone Numbers</label>
                <hr style="border:0; border-top:1px solid #334155; margin:15px 0;">
                <label>UI Opacity</label>
                <input type="range" id="range-op" min="0.3" max="1" step="0.05" value="${config.opacity}" style="width:100%;">
                <button class="btn btn-red" id="btn-wipe" style="width:100%; margin-top:20px;">PURGE ALL DATA</button>
            </div>
        </div>
        <div id="osayuki-resize"></div>
    `;
    document.body.appendChild(root);

    // --- INTERACTION LOGIC (DRAG/RESIZE/TOUCH) ---
    let activeAction = null, startX, startY, startW, startH, startT, startL;

    const startMove = (e) => {
        activeAction = 'drag';
        let ev = e.touches ? e.touches[0] : e;
        startX = ev.clientX; startY = ev.clientY;
        startT = root.offsetTop; startL = root.offsetLeft;
    };

    const startResize = (e) => {
        activeAction = 'resize';
        let ev = e.touches ? e.touches[0] : e;
        startX = ev.clientX; startY = ev.clientY;
        startW = root.offsetWidth; startH = root.offsetHeight;
        e.stopPropagation(); e.preventDefault();
    };

    const doAction = (e) => {
        if (!activeAction) return;
        let ev = e.touches ? e.touches[0] : e;
        if (activeAction === 'drag') {
            root.style.top = (startT + ev.clientY - startY) + 'px';
            root.style.left = (startL + ev.clientX - startX) + 'px';
        } else if (activeAction === 'resize') {
            root.style.width = (startW + ev.clientX - startX) + 'px';
            root.style.height = (startH + ev.clientY - startY) + 'px';
        }
    };

    const stopAction = () => {
        if (activeAction) {
            config.pos = { t: root.style.top, l: root.style.left };
            config.size = { w: root.style.width, h: root.style.height };
            GM_setValue("scraper_config", JSON.stringify(config));
        }
        activeAction = null;
    };

    document.getElementById('osayuki-header').onmousedown = startMove;
    document.getElementById('osayuki-header').ontouchstart = startMove;
    document.getElementById('osayuki-resize').onmousedown = startResize;
    document.getElementById('osayuki-resize').ontouchstart = startResize;
    window.onmousemove = doAction; window.ontouchmove = doAction;
    window.onmouseup = stopAction; window.ontouchend = stopAction;

    // --- NAVIGATION ---
    document.querySelectorAll('.nav-tab').forEach(tab => {
        tab.onclick = () => {
            document.querySelectorAll('.nav-tab').forEach(t => t.classList.remove('active'));
            document.querySelectorAll('.tab-page').forEach(p => p.classList.add('hidden'));
            tab.classList.add('active');
            document.getElementById('p-' + tab.dataset.page).classList.remove('hidden');
            updateDisplay();
        };
    });

    // --- SCRAPING ENGINE ---
    const scrape = () => {
        const text = document.body.innerText;
        let updated = false;
        if (config.emails) {
            const matches = text.match(/[a-zA-Z0-9._%+-]+@(gmail\.com|outlook\.com|hotmail\.com)/gi);
            if (matches) matches.forEach(m => {
                const email = m.toLowerCase();
                if (!leads.find(l => l.val === email)) {
                    leads.push({ type: 'Email', val: email, url: window.location.href, date: new Date().toLocaleDateString() });
                    updated = true;
                }
            });
        }
        if (updated) {
            GM_setValue("leads", JSON.stringify(leads));
            updateDisplay();
            if (leads.length % 100 === 0) autoDownload();
        }
    };

    function updateDisplay() {
        document.getElementById('count-total').innerText = leads.length;
        document.getElementById('p-emails').innerHTML = leads.filter(l => l.type === 'Email').map(l => `
            <div class="lead-card">
                <div style="font-size:12px; font-weight:bold;">${l.val}</div>
                <div style="font-size:10px; color:#94a3b8;">${l.date}</div>
                <button class="copy-btn" onclick="navigator.clipboard.writeText('${l.val}')">Copy</button>
            </div>`).join('');
    }

    // --- SETTINGS & BUTTONS ---
    document.getElementById('range-op').oninput = (e) => {
        config.opacity = e.target.value;
        root.style.background = `rgba(15, 23, 42, ${config.opacity})`;
        GM_setValue("scraper_config", JSON.stringify(config));
    };

    document.getElementById('btn-copy-all').onclick = () => {
        const all = leads.filter(l => l.type === 'Email').map(l => l.val).join('\n');
        navigator.clipboard.writeText(all);
        alert("Copied all emails!");
    };

    document.getElementById('btn-wipe').onclick = () => {
        if (prompt("Type 'CONFIRM' to wipe data") === 'CONFIRM') {
            leads = []; GM_setValue("leads", "[]"); updateDisplay();
        }
    };

    document.getElementById('os-close').onclick = () => root.classList.add('hidden');
    document.addEventListener('keydown', (e) => { if(e.altKey && e.key.toLowerCase() === 'e') root.classList.toggle('hidden'); });

    setInterval(scrape, 3000);
    updateDisplay();
})();
