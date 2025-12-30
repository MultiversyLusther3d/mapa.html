<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Ateliê de Cartografia - Cristino</title>
    <style>
        :root { --cor-papel: #f4e4bc; --cor-tinta: #1a1a40; }
        body { margin: 0; overflow: hidden; background: #1a1b1e; font-family: 'Georgia', serif; touch-action: none; }
        
        /* Menu Lateral */
        #menu-lateral { 
            position: absolute; top: 0; left: 0; width: 280px; height: 100%; 
            background: #2b2d30; color: #adb5bd; padding: 25px; 
            z-index: 100; border-right: 1px solid #d3af37; 
            transition: transform 0.5s cubic-bezier(0.4, 0, 0.2, 1); 
            box-shadow: 10px 0 30px rgba(0,0,0,0.5);
            overflow-y: auto;
        }
        .recolhido { transform: translateX(-100%); }

        h2 { color: #fff; border-bottom: 1px solid #494c52; padding-bottom: 10px; font-size: 1.2em; }
        .campo { margin-bottom: 20px; }
        label { display: block; font-size: 0.8em; color: #d3af37; margin-bottom: 8px; }
        
        button { 
            width: 100%; padding: 12px; border-radius: 4px; border: none; 
            font-weight: bold; cursor: pointer; margin-top: 10px; transition: 0.3s;
        }
        .btn-primario { background: #d3af37; color: #1a1a1a; }
        .btn-primario:hover { background: #fff; }
        
        .seletor-ferramenta { display: flex; gap: 5px; margin-bottom: 10px; }
        .tool-btn { background: #1e1f22; color: #888; border: 1px solid #494c52; flex: 1; }
        .tool-btn.ativo { background: #3574f0; color: white; border-color: #3574f0; }

        /* Área do Papel */
        #canvas-wrap { 
            width: 100vw; height: 100vh; display: flex; justify-content: center; 
            align-items: center; background: #121212; 
        }
        canvas { 
            background: var(--cor-papel); cursor: crosshair; 
            box-shadow: 0 0 50px rgba(0,0,0,0.8);
            background-image: url('https://www.transparenttextures.com/patterns/old-map.png');
            touch-action: none;
        }
        .modo-botao { cursor: pointer !important; outline: 3px solid #d3af37; }

        /* Folha de Anotação */
        #folha-anotacao { 
            display: none; position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%);
            width: 650px; height: 75vh; background: #fdf6e3; border: 10px solid #2c1e0a;
            z-index: 200; padding: 40px; box-shadow: 0 0 150px rgba(0,0,0,1);
        }
        textarea { 
            width: 100%; height: 70%; background: transparent; border: none; 
            font-size: 1.2em; resize: none; outline: none; font-family: 'Courier New', monospace;
            border-top: 1px solid rgba(0,0,0,0.1); padding-top: 20px;
        }

        .btn-toggle-menu { position: fixed; top: 20px; left: 20px; z-index: 150; padding: 10px 15px; background: #2b2d30; color: #d3af37; border: 1px solid #d3af37; border-radius: 4px; cursor: pointer; font-weight: bold; }
    </style>
</head>
<body>

<button class="btn-toggle-menu" onclick="toggleMenu()">☰ CONFIGURAR</button>

<div id="menu-lateral">
    <h2>Ateliê de Cartografia</h2>
    
    <div class="campo">
        <label>Ferramenta:</label>
        <div class="seletor-ferramenta">
            <button id="btn-tinta" class="tool-btn ativo" onclick="setFerramenta('tinta')">Tinta</button>
            <button id="btn-borracha" class="tool-btn" onclick="setFerramenta('borracha')">Borracha</button>
        </div>
    </div>

    <div class="campo">
        <label>Espessura:</label>
        <input type="range" id="brush-size" min="1" max="25" value="2">
    </div>
    
    <button style="background:#555; color:white;" onclick="limparPapel()">Limpar Tudo</button>
    <button class="btn-primario" onclick="registrarMapa()">REGISTRAR E SALVAR</button>
    
    <div id="status-aviso" style="margin-top:20px; font-size:0.8em; color:#d3af37; display:none; text-align:center;">
        <b>MAPA ATIVO</b><br>Clique no papel para anotar.
    </div>
</div>

<div id="canvas-wrap">
    <canvas id="mapaCanvas" width="1400" height="900"></canvas>
</div>

<div id="folha-anotacao">
    <h2 style="color:#2c1e0a; margin-top:0;">Anotações literária</h2>
    <textarea id="texto-area" placeholder="Descreva aqui a geografia, cidades e perigos deste mapa..."></textarea>
    <div style="display:flex; gap:10px;">
        <button class="btn-primario" onclick="salvarAnotacao()">Salvar Crônica</button>
        <button style="background:#888; color:white; width:auto; padding:0 20px;" onclick="fecharFolha()">Voltar</button>
    </div>
</div>

<script>
    const canvas = document.getElementById('mapaCanvas');
    const ctx = canvas.getContext('2d');
    let desenhando = false;
    let mapaRegistrado = false;
    let ferramenta = 'tinta'; // ou 'borracha'

    // Estilo inicial da linha
    ctx.lineJoin = 'round';
    ctx.lineCap = 'round';
    const corTinta = '#1a1a40';
    const corPapel = '#f4e4bc';

    function toggleMenu() { document.getElementById('menu-lateral').classList.toggle('recolhido'); }

    function setFerramenta(tipo) {
        ferramenta = tipo;
        document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('ativo'));
        document.getElementById('btn-' + tipo).classList.add('ativo');
    }

    // Lógica de Desenho
    function iniciarDesenho(e) {
        if (mapaRegistrado) {
            abrirEscrita();
            return;
        }
        
        // Esconder menu ao começar desenhar
        document.getElementById('menu-lateral').classList.add('recolhido');
        
        desenhando = true;
        ctx.beginPath();
        const pos = getPos(e);
        ctx.moveTo(pos.x, pos.y);
    }

    function desenhar(e) {
        if (!desenhando || mapaRegistrado) return;
        const pos = getPos(e);
        
        ctx.lineWidth = document.getElementById('brush-size').value;
        
        if (ferramenta === 'borracha') {
            ctx.strokeStyle = corPapel; // A borracha usa a cor do papel
            ctx.globalCompositeOperation = 'destination-out'; // Técnica para apagar pixels
        } else {
            ctx.strokeStyle = corTinta;
            ctx.globalCompositeOperation = 'source-over';
        }

        ctx.lineTo(pos.x, pos.y);
        ctx.stroke();
    }

    function pararDesenho() { 
        desenhando = false; 
        ctx.globalCompositeOperation = 'source-over'; // Reseta modo para tinta
    }

    function getPos(e) {
        const rect = canvas.getBoundingClientRect();
        const clientX = e.touches ? e.touches[0].clientX : e.clientX;
        const clientY = e.touches ? e.touches[0].clientY : e.clientY;
        return { x: clientX - rect.left, y: clientY - rect.top };
    }

    // Gerenciamento do Mapa
    function limparPapel() {
        if (confirm("Apagar todo o mapa?")) {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            mapaRegistrado = false;
            canvas.classList.remove('modo-botao');
            document.getElementById('status-aviso').style.display = 'none';
        }
    }

    function registrarMapa() {
        mapaRegistrado = true;
        canvas.classList.add('modo-botao');
        document.getElementById('status-aviso').style.display = 'block';
        toggleMenu(); 
        
        // Salva estado no navegador
        localStorage.setItem('cartografia_cristino_img', canvas.toDataURL());
        alert("Mapa registrado! Agora clique nele para escrever suas anotações.");
    }

    function abrirEscrita() {
        document.getElementById('folha-anotacao').style.display = 'block';
        document.getElementById('texto-area').value = localStorage.getItem('cartografia_cristino_texto') || "";
    }

    function salvarAnotacao() {
        localStorage.setItem('cartografia_cristino_texto', document.getElementById('texto-area').value);
        fecharFolha();
    }

    function fecharFolha() { document.getElementById('folha-anotacao').style.display = 'none'; }

    // Eventos
    canvas.addEventListener('mousedown', iniciarDesenho);
    canvas.addEventListener('mousemove', desenhar);
    window.addEventListener('mouseup', pararDesenho);

    canvas.addEventListener('touchstart', (e) => { e.preventDefault(); iniciarDesenho(e); }, {passive: false});
    canvas.addEventListener('touchmove', (e) => { e.preventDefault(); desenhar(e); }, {passive: false});
    window.addEventListener('touchend', pararDesenho);

    // Recuperação Automática
    window.onload = () => {
        const salvo = localStorage.getItem('cartografia_cristino_img');
        if (salvo) {
            const img = new Image();
            img.onload = () => ctx.drawImage(img, 0, 0);
            img.src = salvo;
        }
    };
</script>
</body>
</html>
