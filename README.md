# FIAP - Faculdade de Informática e Administração Paulista

<p align="center">
  <img src="logo-fiap.png" alt="FIAP" width="40%">
</p>

<br>

# CardioIA — Fase 3: Monitoramento Contínuo (IoT na Saúde)

Projeto acadêmico do curso de IA da FIAP. Nesta fase construímos um protótipo de **monitor cardíaco vestível** com ESP32, cobrindo o ciclo completo de IoT em saúde digital: **captura → processamento → transmissão → visualização → alerta**, integrando Edge, Fog e Cloud Computing.

## Equipe

Trabalho desenvolvido em grupo. Integrantes:

- João Rafael Gonçalves Ramos — RM: 567908
- Letícia Angelim Guerra — RM: 567501
- Matheus Guimarães França — RM: 567144
- Rivando Bezerra Cavalcanti Neto — RM: 568235

A equipe dividiu as tarefas entre os integrantes. Nesta fase (PBL de IoT), a Parte 1 (Edge Computing) está em `Parte-1/` e a Parte 2 (Fog/Cloud) em `Parte-2/`.

## Visão geral

O mesmo dispositivo e circuito atravessam as duas partes. Os sensores são um **DHT22** (temperatura e umidade) e um **botão** que simula os batimentos (BPM), com LEDs de status e uma chave que simula a conectividade.

- **Parte 1 (Edge):** o ESP32 lê os sensores, classifica cada leitura na borda (normal, febre, taquicardia…) e **guarda tudo localmente** (SPIFFS), com resiliência offline e política de armazenamento limitado. Quando há conexão, sincroniza os dados pendentes.
- **Parte 2 (Fog/Cloud):** o ponto em que a Parte 1 enviava para a "nuvem" passa a **publicar via MQTT** (broker HiveMQ). Um **dashboard em tempo real** assina o tópico e mostra gráfico de BPM, medidores (gauges) e **alertas automáticos** quando os limites são ultrapassados (temperatura ≥ 38 °C ou BPM > 120). Inclui ainda visualização avançada no Grafana e a mesma lógica de resiliência (as leituras não se perdem se a conexão cair).

## Links das simulações (Wokwi)

- Parte 1: https://wokwi.com/projects/479420907001327617
- Parte 2: https://wokwi.com/projects/479547665221219329

## Estrutura do repositório

| Caminho | Conteúdo |
|---------|----------|
| `README.md` | Este arquivo |
| `logo-fiap.png` | Logo da FIAP |
| `Parte-1/` | Edge Computing — ESP32, armazenamento local e resiliência |
| `Parte-2/` | Fog/Cloud — transmissão MQTT e dashboard |
| `Parte-2/flows.json` | Dashboard do Node-RED (importável) |
| `Parte-2/relatorio-parte2.md` | Relatório: fluxo MQTT + configuração do dashboard |
| `Parte-2/dashboard-web/` | Visualizador leve usado para gerar os prints |
| `Parte-2/prints/` | Evidências do dashboard funcionando |

Cada parte tem seu próprio `README.md` com os detalhes técnicos, os entregáveis e as instruções de execução.
