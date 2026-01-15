<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Sistema de Inventário HS</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css" />
  <script src="https://unpkg.com/html5-qrcode"></script>
  <style>
    /* Estilos baseados no layout original que você gosta */
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
    .btn-secundario{background:#5f6368;margin-top:10px}
    .hidden{display:none}
    .card{background:#1e1e1e;padding:20px;border-radius:10px;margin-bottom:20px;border:1px solid #333}
    h2{text-align:center;margin-top:0;color:#673ab7}
    
    /* Scanner Overlay */
    #scanner-overlay{
      position:fixed;inset:0;background:rgba(0,0,0,0.9);z-index:1000;
      display:none;flex-direction:column;align-items:center;justify-content:center;
    }
    #reader{width:90%;max-width:400px;border:2px solid #673ab7;border-radius:12px;overflow:hidden}
    .scanner-controls{margin-top:20px;width:90%;max-width:400px}
    
    .status-msg{text-align:center;margin-top:10px;font-size:13px}
    .success{color:#4caf50}
    .error{color:#f44336}
  </style>
</head>
<body>

  <div class="container" id="main-menu">
    <h2>Sistema de Inventário</h2>
    <button onclick="showScreen('master-login')">
      <i class="fas fa-user-shield"></i> Painel Master (Leonardo)
    </button>
    <button class="btn-secundario" onclick="showConferentePanel()">
      <i class="fas fa-barcode"></i> Iniciar Contagem (Conferente)
    </button>
  </div>

  <!-- TELA MASTER LOGIN -->
  <div class="container hidden" id="master-login">
    <h2>Acesso Master</h2>
    <div class="form-group">
      <label>Senha Master</label>
      <input type="password" id="master-pass" placeholder="Digite a senha master" />
    </div>
    <button onclick="doMasterLogin()">Entrar</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Voltar</button>
  </div>

  <!-- TELA MASTER PAINEL -->
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
    <button onclick="doCreateInventory()">Criar Inventário</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Sair</button>
    <div id="master-status" class="status-msg"></div>
  </div>

  <!-- TELA CONFERENTE LOGIN -->
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
    <button onclick="doConferenteLogin()">Iniciar Scanner</button>
    <button class="btn-secundario" onclick="showScreen('main-menu')">Voltar</button>
  </div>

  <!-- TELA DE CONTAGEM ATIVA -->
  <div class="container hidden" id="counting-panel">
    <h2 id="active-inv-title">Inventário</h2>
    <div class="card">
      <p><strong>Conferente:</strong> <span id="display-name"></span></p>
      <p><strong>Seção:</strong> <span id="display-section"></span></p>
    </div>
    <button onclick="startScanner()">
      <i class="fas fa-camera"></i> ABRIR SCANNER
    </button>
    <button class="btn-secundario" style="background:#d32f2f" onclick="doConcluirSecao()">
      <i class="fas fa-check-double"></i> CONCLUIR SEÇÃO
    </button>
    <div id="count-status" class="status-msg"></div>
  </div>

  <!-- SCANNER OVERLAY -->
  <div id="scanner-overlay">
    <div id="reader"></div>
    <div class="scanner-controls">
      <button class="btn-secundario" onclick="stopScanner()">
        <i class="fas fa-times"></i> FECHAR SCANNER
      </button>
    </div>
    <div id="scanner-msg" style="margin-top:15px; font-weight:bold"></div>
  </div>

  <script>
    let currentUser = "";
    let currentSection = "";
    let currentInvId = "";
    let currentInvName = "";
    let html5QrCode = null;

    function showScreen(id) {
      document.querySelectorAll('.container').forEach(c => c.classList.add('hidden'));
      document.getElementById(id).classList.remove('hidden');
    }

    // MASTER LOGIC
    function doMasterLogin() {
      const pass = document.getElementById('master-pass').value;
      google.script.run.withSuccessHandler(res => {
        if(res.success) showScreen('master-panel');
        else alert(res.message);
      }).masterLogin(pass);
    }

    function doCreateInventory() {
      const name = document.getElementById('inv-name').value;
      const pass = document.getElementById('inv-pass').value;
      if(!name || !pass) return alert("Preencha todos os campos");
      
      google.script.run.withSuccessHandler(res => {
        alert("Inventário criado com sucesso!");
        showScreen('main-menu');
      }).criarNovoInventario(name, pass);
    }

    // CONFERENTE LOGIC
    function showConferentePanel() {
      google.script.run.withSuccessHandler(list => {
        const select = document.getElementById('inv-select');
        select.innerHTML = list.map(i => `<option value="${i.id}">${i.nome}</option>`).join('');
        showScreen('conferente-login');
      }).listarInventariosAbertos();
    }

    function doConferenteLogin() {
      const name = document.getElementById('conf-name').value;
      const invId = document.getElementById('inv-select').value;
      const invPass = document.getElementById('conf-inv-pass').value;
      const section = document.getElementById('conf-section').value;
      
      if(!name || !invPass || !section) return alert("Preencha todos os campos");

      google.script.run.withSuccessHandler(res => {
        if(res.success) {
          currentUser = name;
          currentSection = section;
          currentInvId = invId;
          currentInvName = res.nomeInventario;
          
          document.getElementById('display-name').textContent = name;
          document.getElementById('display-section').textContent = section;
          document.getElementById('active-inv-title').textContent = res.nomeInventario;
          showScreen('counting-panel');
        } else {
          alert(res.message);
        }
      }).validarAcessoInventario(invId, invPass);
    }

    // SCANNER LOGIC
    function startScanner() {
      document.getElementById('scanner-overlay').style.display = 'flex';
      html5QrCode = new Html5Qrcode("reader");
      const config = { fps: 10, qrbox: { width: 250, height: 250 } };
      
      html5QrCode.start({ facingMode: "environment" }, config, (decodedText) => {
        processBipe(decodedText);
      });
    }

    function stopScanner() {
      if(html5QrCode) {
        html5QrCode.stop().then(() => {
          document.getElementById('scanner-overlay').style.display = 'none';
        });
      } else {
        document.getElementById('scanner-overlay').style.display = 'none';
      }
    }

    function processBipe(ean) {
      // Trava simples de interface para não bipar repetido enquanto processa
      if(window.isProcessing) return;
      window.isProcessing = true;
      
      const msg = document.getElementById('scanner-msg');
      msg.textContent = "Registrando: " + ean;
      msg.style.color = "#673ab7";

      google.script.run.withSuccessHandler(res => {
        window.isProcessing = false;
        if(res.success) {
          msg.textContent = "OK: " + ean;
          msg.style.color = "#4caf50";
          // Beep ou vibração aqui se quiser
          setTimeout(() => { msg.textContent = ""; }, 1000);
        } else {
          alert("Erro ao registrar: " + res.message);
        }
      }).registrarBipeInventario({
        ean: ean,
        usuario: currentUser,
        secao: currentSection,
        nomeInventario: currentInvName
      });
    }

    function doConcluirSecao() {
      if(!confirm("Deseja concluir esta seção e enviar os dados para o consolidado?")) return;
      
      const status = document.getElementById('count-status');
      status.textContent = "Consolidando dados...";
      status.className = "status-msg";

      google.script.run.withSuccessHandler(res => {
        if(res.success) {
          alert("Seção concluída! " + res.count + " itens consolidados.");
          showScreen('main-menu');
        } else {
          status.textContent = "Erro: " + res.message;
          status.className = "status-msg error";
        }
      }).concluirSecao(currentUser, currentInvName);
    }
  </script>
</body>
</html>
