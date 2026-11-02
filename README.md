# CardioIA — Fase 3: Monitoramento Contínuo (IoT na Saúde)

Projeto acadêmico do curso de IA da FIAP. Nesta fase construímos um protótipo de **monitor cardíaco vestível** com ESP32, cobrindo o ciclo completo de IoT em saúde digital: **captura → processamento → transmissão → visualização → alerta**, integrando Edge, Fog e Cloud Computing.

## Autores (dupla)

| Parte | Responsável | Entrega |
|-------|-------------|---------|
| Parte 1 — Armazenamento e processamento local (Edge Computing) | João Rafael | `Parte-1/` |
| Parte 2 — Transmissão para a nuvem e visualização (Fog/Cloud) | Leticia Guerra — RM 567501 | `Parte-2/` |

## Visão geral

O mesmo dispositivo e circuito atravessam as duas partes. Os sensores são um **DHT22** (temperatura e umidade) e um **botão** que simula os batimentos (BPM), com LEDs de status e uma chave que simula a conectividade.

- **Parte 1 (Edge):** o ESP32 lê os sensores, classifica cada leitura na borda (normal, febre, taquicardia…) e **guarda tudo localmente** (SPIFFS), com resiliência offline e política de armazenamento limitado. Quando há conexão, sincroniza os dados pendentes.
- **Parte 2 (Fog/Cloud):** o ponto em que a Parte 1 enviava para a "nuvem" passa a **publicar via MQTT** (broker HiveMQ). Um **dashboard em tempo real** assina o tópico e mostra gráfico de BPM, medidores (gauges) e **alertas automáticos** quando os limites são ultrapassados (temperatura ≥ 38 °C ou BPM > 120). Inclui ainda visualização avançada no Grafana e a mesma lógica de resiliência (as leituras não se perdem se a conexão cair).

## Links das simulações (Wokwi)

- Parte 1: https://wokwi.com/projects/479420907001327617
- Parte 2: https://wokwi.com/projects/479547665221219329

## Estrutura do repositório

```
.
├── README.md            (este arquivo)
├── Parte-1/             Edge Computing — ESP32, armazenamento local e resiliência
│   ├── README.md
│   ├── sketch.ino
│   ├── diagram.json
│   ├── libraries.txt
│   └── partitions.csv
└── Parte-2/             Fog/Cloud — MQTT e dashboard
    ├── README.md
    ├── sketch.ino
    ├── diagram.json
    ├── libraries.txt
    ├── flows.json               (dashboard do Node-RED, importável)
    ├── relatorio-parte2.md      (relatório: fluxo MQTT + configuração do dashboard)
    ├── GUIA-EXECUCAO.md
    ├── dashboard-web/           (visualizador leve usado para gerar os prints)
    └── prints/                  (evidências do dashboard funcionando)
```

Cada parte tem seu próprio `README.md` com os detalhes técnicos, os entregáveis e as instruções de execução.
