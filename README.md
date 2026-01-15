<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Sistema de Inventário HS</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css" />
  <script src="https://unpkg.com/html5-qrcode"></script>
  <style>
    *{box-sizing:border-box}
    body{
      font-family: Arial, sans-serif;
      background:#000;color:#fff;margin:0;padding:0;
      min-height:100vh;display:flex;flex-direction:column;align-items:center;
    }
    .container{
      width:100%;max-width:600px;background:#2c2c2c;
      padding:30px 20px;border-radius:12px;box-shadow:0 0 10px rgba(0,0,0,.5);
      margin:20px 10px;
    }
    .form-group{margin-bottom:15px}
    label{font-size:14px;font-weight:600;display:block;margin-bottom:6px;color:#ccc}
    input, select{
      width:100%;padding:12px;font-size:14px;border:1px solid #444;border-radius:8px;
      background:#2a2a2a;color:#fff;
    }
    button{
      background:#673ab7;color:#fff;border:none;padding:12px 18px;border-radius:8px;
      font-size:14px;cursor:pointer;font-weight:700;transition:.2s;width:100%;
      display:flex;align-items:center;justify-content:center;gap:10px;
    }
    button:hover{background:#5e35b1}
    button:disabled{background:#444;cursor:not-allowed;opacity:0.7}
    .btn-secundario{background:#5f6368;margin-top:10px}
    .hidden{display:none}
    .card{background:#1e1e1e;padding:20px;border-radius:10px;margin-bottom:20px;border:1px solid #333}
    h2{text-align:center;margin-top:0;color:#673ab7}
    
    #scanner-overlay{
      position:fixed;inset:0;background:rgba(0,0,0,0.9);z-index:1000;
      display:none;flex-direction:column;align-items:center;justify-content:center;
    }
    #reader{width:90%;max-width:400px;border:2px solid #673ab7;border-radius:12px;overflow:hidden}
    .scanner-controls{margin-top:20px;width:90%;max-width:400px}
    
    .status-msg{text-align:center;margin-top:10px;font-size:13px}
    .loading-spinner {
      display: inline-block;
      width: 12px; height: 12px;
      border: 2px solid rgba(255,255,255,.3);
      border-radius: 50%;
      border-top-color: #fff;
      animation: spin 1s ease-in-out infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  </style>
</head>
<body>

  <div class="container" id="main-menu">
    <h2>Sistema de Inventário</h2>
    <button onclick="showScreen('master-login')">
      <i class="fas fa-user-shield"></i> Painel Master (Leonardo)
    </button>
    <button class="btn-secundario" id="btn-open-conf" onclick="showConferentePanel()">
      <i class="fas fa-barcode"></i> Iniciar Contagem (Conferente)
    </button>
  </div>

  <div class="container hidden" id="master-login">
    <h2>Acesso Master</h2>
    <div class="form-group">
      <label>Senha Master</label>
      <input type="password" id="master-pass" placeholder="Digite a senha master" />
    </div>
    <button id="btn-master-login" onclick="doMasterLogin()">Entrar</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Voltar</button>
  </div>

  <div class="container hidden" id="master-panel">
    <h2>Novo Inventário</h2>
    <div class="form-group">
      <label>Nome do Inventário</label>
      <input type="text" id="inv-name" placeholder="Ex: Inventário Verão 2026" />
    </div>
    <div class="form-group">
      <label>Senha para Conferentes</label>
      <input type="text" id="inv-pass" placeholder="Crie uma senha" />
    </div>
    <button id="btn-create-inv" onclick="doCreateInventory()">Criar Inventário</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Sair</button>
  </div>

  <div class="container hidden" id="conferente-login">
    <h2>Acesso Conferente</h2>
    <div class="form-group">
      <label>Seu Nome</label>
      <input type="text" id="conf-name" placeholder="Seu nome" />
    </div>
    <div class="form-group">
      <label>Selecione o Inventário</label>
      <select id="inv-select"></select>
    </div>
    <div class="form-group">
      <label>Senha do Inventário</label>
      <input type="password" id="conf-inv-pass" placeholder="Senha do inventário" />
    </div>
    <div class="form-group">
      <label>Seção / Local</label>
      <input type="text" id="conf-section" placeholder="Ex: Corredor A, Prateleira 1" />
    </div>
    <button id="btn-conf-login" onclick="doConferenteLogin()">Iniciar Scanner</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Voltar</button>
  </div>

  <div class="container hidden" id="counting-panel">
    <h2 id="active-inv-title">Inventário</h2>
    <div class="card">
      <p><strong>Conferente:</strong> <span id="display-name"></span></p>
      <p><strong>Seção:</strong> <span id="display-section"></span></p>
    </div>
    <button onclick="startScanner()">
      <i class="fas fa-camera"></i> ABRIR SCANNER
    </button>
    <button class="btn-secundario" id="btn-concluir" style="background:#d32f2f" onclick="doConcluirSecao()">
      <i class="fas fa-check-double"></i> CONCLUIR SEÇÃO
    </button>
    <div id="count-status" class="status-msg"></div>
  </div>

  <div id="scanner-overlay">
    <div id="reader"></div>
    <div class="scanner-controls">
      <button class="btn-secundario" onclick="stopScanner()">
        <i class="fas fa-times"></i> FECHAR SCANNER
      </button>
    </div>
    <div id="scanner-msg" style="margin-top:15px; font-weight:bold; text-align:center"></div>
  </div>

  <script>
    // COLE O SEU LINK DO GOOGLE APPS SCRIPT AQUI (O QUE TERMINA EM /exec)
    const WEB_APP_URL = "https://script.google.com/macros/s/AKfycbxGS0dxhZtqXY5GBvwKzIxzUYwKnQh8qUHI9BYvstsmWMLuz8uyDwYZmaOA-SsyV4gcVQ/exec";
    
    let currentUser = "";
    let currentSection = "";
    let currentInvId = "";
    let currentInvName = "";
    let html5QrCode = null;

    function showScreen(id) {
      document.querySelectorAll('.container').forEach(c => c.classList.add('hidden'));
      document.getElementById(id).classList.remove('hidden');
    }

    function setBtnLoading(id, isLoading, text) {
      const btn = document.getElementById(id);
      if (isLoading) {
        btn.disabled = true;
        btn.innerHTML = `<div class="loading-spinner"></div> Processando...`;
      } else {
        btn.disabled = false;
        btn.innerHTML = text;
      }
    }

    // Função para chamar o Google Apps Script via Fetch (CORS-friendly)
    async function callGAS(payload) {
      try {
        const response = await fetch(WEB_APP_URL, {
          method: "POST",
          body: JSON.stringify(payload)
        });
        return await response.json();
      } catch (err) {
        console.error("Erro na chamada GAS:", err);
        throw err;
      }
    }

    // MASTER LOGIC
    async function doMasterLogin() {
      const pass = document.getElementById('master-pass').value;
      if(!pass) return alert("Digite a senha");
      setBtnLoading('btn-master-login', true);
      
      try {
        const res = await callGAS({ api: 'masterLogin', senha: pass });
        setBtnLoading('btn-master-login', false, 'Entrar');
        if(res.success) showScreen('master-panel');
        else alert(res.message);
      } catch (e) {
        setBtnLoading('btn-master-login', false, 'Entrar');
        alert("Erro ao conectar com o Google Script. Verifique se o link está correto e se você fez o Deploy como 'Qualquer pessoa'.");
      }
    }

    async function doCreateInventory() {
      const name = document.getElementById('inv-name').value;
      const pass = document.getElementById('inv-pass').value;
      if(!name || !pass) return alert("Preencha todos os campos");
      setBtnLoading('btn-create-inv', true);
      
      try {
        const res = await callGAS({ api: 'criarNovoInventario', nome: name, senha: pass });
        setBtnLoading('btn-create-inv', false, 'Criar Inventário');
        if(res.success) { alert("Inventário criado!"); showScreen('main-menu'); }
        else alert(res.message);
      } catch (e) {
        setBtnLoading('btn-create-inv', false, 'Criar Inventário');
        alert("Erro ao criar inventário.");
      }
    }

    async function showConferentePanel() {
      setBtnLoading('btn-open-conf', true);
      try {
        // Para listar usamos GET para ser mais simples
        const resp = await fetch(WEB_APP_URL + "?api=listarInventariosAbertos");
        const list = await resp.json();
        
        setBtnLoading('btn-open-conf', false, '<i class="fas fa-barcode"></i> Iniciar Contagem (Conferente)');
        const select = document.getElementById('inv-select');
        if(!list || list.length === 0) return alert("Não há inventários abertos.");
        select.innerHTML = list.map(i => `<option value="${i.id}">${i.nome}</option>`).join('');
        showScreen('conferente-login');
      } catch (e) {
        setBtnLoading('btn-open-conf', false, '<i class="fas fa-barcode"></i> Iniciar Contagem (Conferente)');
        alert("Erro ao buscar inventários.");
      }
    }

    async function doConferenteLogin() {
      const name = document.getElementById('conf-name').value;
      const invId = document.getElementById('inv-select').value;
      const invPass = document.getElementById('conf-inv-pass').value;
      const section = document.getElementById('conf-section').value;
      if(!name || !invPass || !section) return alert("Preencha todos os campos");
      setBtnLoading('btn-conf-login', true);
      
      try {
        const res = await callGAS({ api: 'validarAcessoInventario', idInventario: invId, senhaDigitada: invPass });
        setBtnLoading('btn-conf-login', false, 'Iniciar Scanner');
        if(res.success) {
          currentUser = name; currentSection = section; currentInvId = invId; currentInvName = res.nomeInventario;
          document.getElementById('display-name').textContent = name;
          document.getElementById('display-section').textContent = section;
          document.getElementById('active-inv-title').textContent = res.nomeInventario;
          showScreen('counting-panel');
        } else alert(res.message);
      } catch (e) {
        setBtnLoading('btn-conf-login', false, 'Iniciar Scanner');
        alert("Erro ao validar acesso.");
      }
    }

    function startScanner() {
      document.getElementById('scanner-overlay').style.display = 'flex';
      html5QrCode = new Html5Qrcode("reader");
      html5QrCode.start({ facingMode: "environment" }, { fps: 10, qrbox: 250 }, processBipe);
    }

    function stopScanner() {
      if(html5QrCode) html5QrCode.stop().then(() => { document.getElementById('scanner-overlay').style.display = 'none'; });
      else document.getElementById('scanner-overlay').style.display = 'none';
    }

    async function processBipe(ean) {
      if(window.isProcessing) return;
      window.isProcessing = true;
      const msg = document.getElementById('scanner-msg');
      msg.textContent = "Registrando: " + ean;
      msg.style.color = "#673ab7";

      try {
        const res = await callGAS({ api: 'registrarBipeInventario', ean, usuario: currentUser, secao: currentSection, nomeInventario: currentInvName });
        window.isProcessing = false;
        if(res.success) {
          msg.textContent = "✅ OK: " + ean; msg.style.color = "#4caf50";
          if(navigator.vibrate) navigator.vibrate(50);
          setTimeout(() => { msg.textContent = ""; }, 1000);
        } else alert("Erro: " + res.message);
      } catch (e) {
        window.isProcessing = false;
        alert("Erro de conexão ao registrar bipe.");
      }
    }

    async function doConcluirSecao() {
      if(!confirm("Deseja concluir esta seção?")) return;
      setBtnLoading('btn-concluir', true);
      try {
        const res = await callGAS({ api: 'concluirSecao', usuario: currentUser, nomeInventario: currentInvName });
        setBtnLoading('btn-concluir', false, '<i class="fas fa-check-double"></i> CONCLUIR SEÇÃO');
        if(res.success) { alert("Seção concluída! " + res.count + " itens."); showScreen('main-menu'); }
        else alert("Erro: " + res.message);
      } catch (e) {
        setBtnLoading('btn-concluir', false, '<i class="fas fa-check-double"></i> CONCLUIR SEÇÃO');
        alert("Erro ao concluir seção.");
      }
    }
  </script>
</body>
</html>
