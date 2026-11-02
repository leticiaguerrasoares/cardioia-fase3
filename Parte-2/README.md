# CardioIA — Fase 3, Parte 2: Transmissão para a nuvem e visualização (Fog/Cloud)

**Aluna:** Leticia Guerra — RM 567501

Continuação da Parte 1 (Edge Computing). Aqui o monitor cardíaco vestível deixa de só guardar os dados na borda e passa a **enviá-los para a nuvem via MQTT** e a **mostrá-los em tempo real** num dashboard (Node-RED e, opcionalmente, Grafana), com medidores e alertas automáticos.

**Simulação no Wokwi:** https://wokwi.com/projects/479547665221219329

## O que foi entregue

- `sketch.ino` — código C++ do ESP32, comentado, publicando as leituras via MQTT.
- `diagram.json` — circuito do Wokwi (o mesmo da Parte 1: DHT22, botão de batimento, chave de conectividade e LEDs).
- `libraries.txt` — bibliotecas do Wokwi (DHT ESPx e PubSubClient).
- `flows.json` — **export do dashboard do Node-RED** (gráfico de BPM, gauges de temperatura e BPM, indicador de alerta). Já conta como um dos entregáveis.
- `relatorio-parte2.md` — relatório sobre o fluxo MQTT e a configuração do dashboard.
- `GUIA-EXECUCAO.md` — passo a passo para rodar tudo e tirar os prints, incluindo o Grafana.
- `prints/` — pasta para as capturas de tela do dashboard funcionando.

## Resumo técnico

| Item | Valor |
|------|-------|
| Sensores | DHT22 (temperatura/umidade) + botão (BPM) |
| Protocolo | MQTT |
| Broker | `broker.hivemq.com` : `1883` (público) |
| Tópico de telemetria | `cardioia/leticia567501/telemetria` |
| Tópico de alerta | `cardioia/leticia567501/alerta` (retido) |
| Dashboard | Node-RED (node-red-dashboard) + Grafana (opcional) |
| Limites de alerta | temperatura ≥ 38 °C · BPM > 120 |

## Como funciona, em uma frase

O ESP32 lê os sensores, classifica a leitura na borda, publica um JSON no broker MQTT; o dashboard assina esse tópico e mostra gráfico, medidores e um indicador que fica vermelho quando há alerta. Se a conexão cair, as leituras ficam numa fila e são reenviadas quando a conexão volta.

## Evidências (pasta `prints/`)

- **serial_mqtt.png** — Monitor Serial do Wokwi publicando as linhas `[NUVEM] {...}` (dados indo para a nuvem via MQTT).
- **dashboard_normal.png** — dashboard em estado normal (faixa verde, temperatura 36,5 °C).
- **dashboard_alerta_febre.png** — dashboard em alerta de febre (faixa vermelha, medidor de temperatura na zona vermelha a 39,2 °C).
- **dashboard_alerta_bpm.png** — dashboard com o gráfico de BPM em tempo real e alerta combinado (febre + bradicardia).

O arquivo `dashboard-web/dashboard.html` é um visualizador leve que conecta no mesmo broker por WebSocket, usado para gerar esses prints sem instalar nada. A dashboard pedida no enunciado (Node-RED) está no `flows.json`.

Para rodar e gerar as evidências, siga o `GUIA-EXECUCAO.md`.
