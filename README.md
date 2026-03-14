# App-scanner
Proyecto que se conlleva a escanear animales de todo tipo amenaza y cuanto peligro causa 
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bio-Scanner | Ultra-Fast AI</title>
    
    <!-- Librerías Optimizadas -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/mobilenet@latest"></script>
    <script async src="https://docs.opencv.org/4.5.4/opencv.js"></script>
    
    <style>
        :root {
            --neon: #00ffcc;
            --danger: #ff0044;
            --bg: #030303;
            --card: #0a0a0c;
        }

        body {
            font-family: 'Consolas', monospace;
            background: var(--bg);
            color: #fff;
            margin: 0;
            overflow: hidden;
            display: flex;
            flex-direction: column;
            height: 100vh;
        }

        /* HEADER */
        header {
            padding: 8px 20px;
            background: #000;
            border-bottom: 1px solid #1a1a1a;
            display: flex;
            justify-content: space-between;
            font-size: 0.7rem;
            color: #444;
        }

        .active { color: var(--neon); text-shadow: 0 0 5px var(--neon); }

        /* LAYOUT */
        main {
            display: grid;
            grid-template-columns: 1fr 400px;
            flex-grow: 1;
        }

        /* VISOR IZQUIERDO */
        .visor {
            position: relative;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            background: radial-gradient(circle, #0a0e14 0%, #030303 100%);
            border-right: 1px solid #111;
        }

        .canvas-container {
            position: relative;
            border: 1px solid #222;
            background: #000;
            width: 90%;
            height: 70%;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        canvas { max-width: 100%; max-height: 100%; }

        .scan-bar {
            position: absolute;
            width: 100%;
            height: 2px;
            background: var(--neon);
            box-shadow: 0 0 15px var(--neon);
            top: 0;
            display: none;
            animation: fastScan 1.5s infinite linear;
            z-index: 10;
        }

        @keyframes fastScan { from { top: 0; } to { top: 100%; } }

        /* PANEL DERECHO */
        .panel {
            background: var(--card);
            padding: 20px;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        .stat-box {
            background: #000;
            padding: 15px;
            border: 1px solid #1a1a1a;
            border-left: 4px solid #333;
        }

        .val { font-size: 2.2rem; font-weight: bold; margin: 5px 0; line-height: 1; }
        .desc { font-size: 0.6rem; color: #555; text-transform: uppercase; }

        .progress-bg { height: 4px; background: #111; margin-top: 10px; }
        .progress-fill { height: 100%; width: 0%; transition: width 0.4s ease-out; }

        #report {
            flex-grow: 1;
            font-size: 0.8rem;
            color: #888;
            background: #000;
            padding: 12px;
            border: 1px solid #111;
            overflow-y: auto;
        }

        .btn {
            background: var(--neon);
            color: #000;
            border: none;
            padding: 15px;
            font-weight: bold;
            cursor: pointer;
            text-transform: uppercase;
        }

        .btn:disabled { background: #222; color: #444; cursor: wait; }
    </style>
</head>
<body>

    <header>
        <div>CORE_v9: BIO-ACCELERATED</div>
        <div>
            STATUS: 
            <span id="st-cv">OPENCV</span> | 
            <span id="st-tf">TF.JS</span> | 
            <span id="st-ol">OLLAMA</span>
        </div>
    </header>

    <main>
        <section class="visor">
            <div class="canvas-container">
                <div id="scan-bar" class="scan-bar"></div>
                <canvas id="out-cv"></canvas>
            </div>
            <input type="file" id="file-in" hidden accept="image/*">
            <button id="btn-run" class="btn" disabled style="margin-top:20px; width:90%">Cargando Motores...</button>
        </section>

        <aside class="panel">
            <div class="stat-box" id="box-species">
                <div class="desc">Especie Detectada</div>
                <div id="id-name" style="font-size: 1.1rem; color: var(--neon);">---</div>
            </div>

            <div class="stat-box" id="box-attack">
                <div class="desc">Probabilidad de Ataque Cercano</div>
                <div id="val-attack" class="val" style="color: var(--danger);">0%</div>
                <div class="progress-bg"><div id="bar-attack" class="progress-fill" style="background: var(--danger);"></div></div>
            </div>

            <div class="stat-box" id="box-threat">
                <div class="desc">Índice de Amenaza General</div>
                <div id="val-threat" class="val" style="color: #ffaa00;">0%</div>
                <div class="progress-bg"><div id="bar-threat" class="progress-fill" style="background: #ffaa00;"></div></div>
            </div>

            <div id="report">
                [SISTEMA LISTO] Esperando entrada de datos biológicos...
            </div>
        </aside>
    </main>

    <img id="buffer" style="display:none">

    <script>
        let model;
        const btn = document.getElementById('btn-run');
        
        // --- DATA NATIVA (RESULTADOS INSTANTÁNEOS) ---
        const bioData = {
            "lion": [95, 85], "tiger": [97, 90], "bear": [92, 70], "snake": [88, 95], "cobra": [98, 98],
            "shark": [94, 60], "crocodile": [96, 98], "dog": [15, 20], "cat": [5, 5], "spider": [60, 40],
            "hippo": [98, 95], "gorilla": [80, 40], "wasp": [20, 90], "eagle": [40, 30], "wolf": [85, 75]
        };

        // --- INICIO ACELERADO ---
        (async function init() {
            model = await mobilenet.load();
            document.getElementById('st-tf').className = 'active';
            
            const waitCV = setInterval(() => {
                if (window.cv && cv.Mat) {
                    document.getElementById('st-cv').className = 'active';
                    btn.disabled = false;
                    btn.innerText = "ESCANEAR AHORA";
                    clearInterval(waitCV);
                    checkOllama();
                }
            }, 100);
        })();

        async function checkOllama() {
            try {
                const r = await fetch('http://localhost:11434/api/tags');
                if (r.ok) document.getElementById('st-ol').className = 'active';
            } catch(e) {}
        }

        // --- PROCESAMIENTO FLASH ---
        btn.onclick = () => document.getElementById('file-in').click();

        document.getElementById('file-in').onchange = (e) => {
            const file = e.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (ev) => {
                const img = document.getElementById('buffer');
                img.src = ev.target.result;
                img.onload = () => process(img);
            };
            reader.readAsDataURL(file);
        };

        async function process(img) {
            document.getElementById('scan-bar').style.display = 'block';
            
            // 1. OpenCV (Filtro rápido)
            let src = cv.imread(img);
            let dst = new cv.Mat();
            cv.cvtColor(src, src, cv.COLOR_RGBA2GRAY, 0);
            cv.Canny(src, dst, 50, 150, 3, false);
            cv.imshow('out-cv', dst);
            src.delete(); dst.delete();

            // 2. TF.js (Identificación)
            const pred = await model.classify(img);
            const name = pred[0].className.split(',')[0].toLowerCase();
            document.getElementById('id-name').innerText = name.toUpperCase();

            // 3. Resultado Instantáneo (Base de Datos)
            let scores = [10, 5]; // Default
            for (let key in bioData) {
                if (name.includes(key)) {
                    scores = bioData[key];
                    break;
                }
            }
            fastUI(scores[0], scores[1]);

            // 4. Ollama (Background Report - No bloquea la UI)
            generateReport(name, scores[0], scores[1]);
        }

        function fastUI(threat, attack) {
            document.getElementById('val-threat').innerText = threat + "%";
            document.getElementById('bar-threat').style.width = threat + "%";
            document.getElementById('val-attack').innerText = attack + "%";
            document.getElementById('bar-attack').style.width = attack + "%";
        }

        async function generateReport(name, t, a) {
            const report = document.getElementById('report');
            report.innerHTML = "Analizando comportamiento detallado...";
            
            try {
                const response = await fetch('http://localhost:11434/api/generate', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        model: "llama3.2",
                        prompt: `Animal: ${name}. Peligro: ${t}%. Ataque: ${a}%. Explica por qué es peligroso brevemente.`,
                        stream: true
                    })
                });

                const reader = response.body.getReader();
                const decoder = new TextDecoder();
                let text = "";
                report.innerHTML = "";

                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;
                    const json = JSON.parse(decoder.decode(value));
                    if (json.response) {
                        text += json.response;
                        report.innerHTML = text.replace(/\n/g, '<br>');
                    }
                }
            } catch (e) {
                report.innerHTML = "[INFO LOCAL]: Datos basados en parámetros biológicos de la base de datos interna.";
            }
            document.getElementById('scan-bar').style.display = 'none';
        }
    </script>
</body>
</html>
