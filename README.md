<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EcoTrânsito - Ilhas de Calor & Estresse Urbano</title>
    <!-- Leaflet CSS para o Mapa -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        :root {
            --primary: #2c3e50;
            --bg: #f4f6f7;
            --card-bg: #ffffff;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            padding: 0;
            background-color: var(--bg);
            color: #333;
            display: flex;
            flex-direction: column;
            height: 100vh;
        }
        header {
            background-color: var(--primary);
            color: white;
            padding: 15px;
            text-align: center;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        header h1 { margin: 0; font-size: 1.4rem; }
        header p { margin: 5px 0 0 0; font-size: 0.9rem; opacity: 0.8; }
        
        .container {
            display: flex;
            flex: 1;
            overflow: hidden;
        }
        #map {
            flex: 1;
            height: 100%;
        }
        .panel {
            width: 380px;
            background: var(--card-bg);
            box-shadow: -2px 0 10px rgba(0,0,0,0.1);
            padding: 20px;
            display: flex;
            flex-direction: column;
            gap: 15px;
            overflow-y: auto;
            z-index: 1000;
        }
        .card {
            background: #f8f9fa;
            border-left: 5px solid #bdc3c7;
            padding: 15px;
            border-radius: 4px;
        }
        .card.danger { border-left-color: #e74c3c; }
        .card.warning { border-left-color: #f39c12; }
        .card.success { border-left-color: #2ecc71; }
        
        .card h3 { margin: 0 0 8px 0; font-size: 1rem; color: var(--primary); }
        .metric { font-size: 1.8rem; font-weight: bold; margin: 5px 0; }
        
        label { font-weight: bold; font-size: 0.9rem; }
        select {
            width: 100%;
            padding: 8px;
            border-radius: 4px;
            border: 1px solid #ccc;
            margin-top: 5px;
        }
        .progress-bar {
            background: #e0e0e0;
            border-radius: 10px;
            height: 15px;
            width: 100%;
            overflow: hidden;
            margin-top: 5px;
        }
        .progress-fill {
            height: 100%;
            width: 0%;
            transition: width 0.5s ease, background-color 0.5s ease;
        }
        @media (max-width: 768px) {
            .container { flex-direction: column; }
            .panel { width: auto; height: 40%; }
            #map { height: 60%; }
        }
    </style>
</head>
<body>

    <header>
        <h1>EcoTrânsito: Monitor de Microclima e Irritabilidade</h1>
        <p>Estudo de Caso: Campina Grande e Massaranduba - PB (Protótipo Didático)</p>
    </header>

    <div class="container">
        <!-- Div do Mapa -->
        <div id="map"></div>

        <!-- Painel Lateral -->
        <div class="panel">
            <h2>Análise de Impacto</h2>
            <p style="font-size: 0.85rem; color: #666;">Selecione uma cidade no mapa e altere o nível do tráfego para simular o estresse biometeorológico.</p>
            
            <div>
                <label for="traffic">Condição do Trânsito Local:</label>
                <select id="traffic" onchange="updateCalculations()">
                    <option value="1">Fluido (Sem retenção)</option>
                    <option value="1.3">Moderado (Horário de pico leve)</option>
                    <option value="1.7">Intenso (Trânsito parado / Congestionamento)</option>
                </select>
            </div>

            <hr style="border: 0; border-top: 1px solid #eee; margin: 10px 0;">

            <div class="card success" id="temp-card">
                <h3>Temperatura Estimada no Asfalto</h3>
                <div class="metric" id="display-temp">-- °C</div>
                <span id="display-microclimate" style="font-size: 0.85rem; color: #555;">Clique em um marcador no mapa</span>
            </div>

            <div class="card" id="stress-card">
                <h3>Índice de Irritabilidade do Motorista</h3>
                <div class="metric" id="display-stress">-- %</div>
                <div class="progress-bar">
                    <div id="stress-bar" class="progress-fill"></div>
                </div>
                <p id="display-behavior" style="font-size: 0.85rem; margin: 8px 0 0 0; font-weight: 500;"></p>
            </div>
        </div>
    </div>

    <!-- Scripts Leaflet para o Mapa -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // Inicializa o mapa focado na região entre Campina Grande e Massaranduba
        const map = L.map('map').setView([-7.21, -35.85], 11);

        // Adiciona a camada visual de mapa (OpenStreetMap)
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '© OpenStreetMap contributors'
        }).addTo(map);

        // Dados estáticos simulados (que substituem a API se necessário, para rodar localmente sem travar)
        const locations = {
            cg_centro: {
                name: "Centro de Campina Grande (Área Crítica)",
                coords: [-7.2241, -35.8819],
                baseTemp: 29.5, // Em um cenário real, você buscaria isso da API
                islandEffect: 3.8, // +3.8°C devido ao asfalto/verticalização
                desc: "Zona de alta retenção térmica (efeito canhão urbano) e alta densidade de veículos."
            },
            massaranduba: {
                name: "Massaranduba - Centro",
                coords: [-7.1952, -35.7891],
                baseTemp: 28.0,
                islandEffect: 0.8, // Quase sem ilha de calor severa devido à vegetação envolvente
                desc: "Zona periférica/rural com boa dissipação de calor e menor retenção no pavimento."
            }
        };

        let activeLocation = null;

        // Adiciona marcadores no mapa
        const markerCG = L.marker(locations.cg_centro.coords).addTo(map)
            .bindPopup(`<b>${locations.cg_centro.name}</b><br>Clique para analisar.`);
        
        const markerMassara = L.marker(locations.massaranduba.coords).addTo(map)
            .bindPopup(`<b>${locations.massaranduba.name}</b><br>Clique para analisar.`);

        // Eventos de clique nos marcadores
        markerCG.on('click', () => selectLocation('cg_centro'));
        markerMassara.on('click', () => selectLocation('massaranduba'));

        function selectLocation(key) {
            activeLocation = locations[key];
            updateCalculations();
        }

        function updateCalculations() {
            if (!activeLocation) return;

            const trafficFactor = parseFloat(document.getElementById('traffic').value);
            
            // Lógica do Algoritmo do seu TCC
            // Temperatura final calculada = Temp Ambiente + Fator de Ilha de Calor Urbano
            const finalTemp = activeLocation.baseTemp + activeLocation.islandEffect;
            
            // O estresse é potencializado se a temperatura passar de 30°C e multiplicado pelo fator de trânsito
            let stressBase = (finalTemp - 25) * 8; 
            let finalStress = Math.min(Math.max(Math.round(stressBase * trafficFactor), 10), 100);

            // Atualiza a Interface do Usuário (UI)
            document.getElementById('display-temp').innerText = `${finalTemp.toFixed(1)} °C`;
            document.getElementById('display-microclimate').innerText = `${activeLocation.name}. ${activeLocation.desc} (Efeito Ilha: +${activeLocation.islandEffect}°C)`;
            
            document.getElementById('display-stress').innerText = `${finalStress}%`;
            
            // Controle da Barra de Progresso e Alertas Visuais
            const stressBar = document.getElementById('stress-bar');
            const stressCard = document.getElementById('stress-card');
            const tempCard = document.getElementById('temp-card');
            const behaviorText = document.getElementById('display-behavior');
            
            stressBar.style.width = `${finalStress}%`;

            // Reset classes
            stressCard.className = "card";
            tempCard.className = "card";

            if (finalStress < 45) {
                stressBar.style.backgroundColor = "#2ecc71";
                stressCard.classList.add("success");
                behaviorText.innerText = "Motorista Calmo: Conforto térmico aceitável. Respostas e reflexos normais frente às adversidades do tráfego.";
                behaviorText.style.color = "#27ae60";
            } else if (finalStress >= 45 && finalStress < 75) {
                stressBar.style.backgroundColor = "#f39c12";
                stressCard.classList.add("warning");
                behaviorText.innerText = "Motorista Incomodado: Fadiga térmica inicial. Propensão a pequenos atos de impaciência (buzinas excessivas, reclamações).";
                behaviorText.style.color = "#d35400";
            } else {
                stressBar.style.backgroundColor = "#e74c3c";
                stressCard.classList.add("danger");
                tempCard.className = "card danger";
                behaviorText.innerText = "ALERTA CRÍTICO: Irritabilidade extrema. Alta propensão a condutas agressivas, discussões e perda de reflexos defensivos devido ao calor sufocante e confinamento.";
                behaviorText.style.color = "#c0392b";
            }
        }
    </script>
</body>
</html>
