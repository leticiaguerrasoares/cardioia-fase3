# CardioIA — Fase 3, Parte 2: Transmissão para a nuvem e visualização
## Relatório sobre o fluxo de comunicação MQTT e a configuração do dashboard

**Aluna:** Leticia Guerra — RM 567501
**Fase 3 — Monitoramento Contínuo: IoT na Saúde (Parte 2 — Fog/Cloud Computing).**

Este relatório descreve como o protótipo do monitor cardíaco vestível, que na Parte 1 (feita pelo João) já lia os sensores e guardava os dados na borda, passa agora a **transmitir essas leituras para a nuvem via MQTT** e a **exibi-las em um dashboard em tempo real** no Node-RED, com medidores e alertas automáticos. A parte visual avançada é complementada pelo Grafana.

---

## 1. Visão geral da arquitetura

O sistema segue o caminho completo que a fase pede: **captura → processamento → transmissão → visualização → alerta**, distribuído em três camadas.

Na **borda (Edge)**, o ESP32 lê o DHT22 (temperatura e umidade) e o botão que simula os batimentos, calcula o BPM e classifica cada leitura como normal ou de alerta. Essa decisão acontece no próprio dispositivo, sem depender da internet — o que é essencial em saúde, porque o LED de alerta precisa acender na hora.

Na camada de **névoa/nuvem (Fog/Cloud)**, as leituras saem do ESP32 pelo protocolo MQTT até um broker (o HiveMQ público). O broker funciona como um "correio" central: o dispositivo publica as mensagens e qualquer ferramenta interessada se inscreve para recebê-las.

Na **visualização**, o Node-RED se inscreve no mesmo tópico, recebe as mensagens e as transforma em gráfico, medidores (gauges) e um indicador de alerta. O Grafana entra como opção de visualização mais rica, consumindo os mesmos dados.

```
[ESP32 + sensores]  --publica-->  [Broker MQTT: HiveMQ]  --assina-->  [Node-RED / Grafana]
     (Edge)                             (Cloud)                        (Visualização)
```

---

## 2. Por que MQTT

MQTT é o protocolo mais usado em IoT, e por bons motivos num cenário como o nosso. Ele é **leve** (as mensagens têm pouquíssimo cabeçalho, o que importa num dispositivo de bateria como um vestível), funciona no modelo **publish/subscribe** (o dispositivo não precisa saber quem vai ler os dados — só publica num "tópico", e quem quiser se inscreve) e foi feito para **redes instáveis**, justamente o caso de um paciente monitorado em casa.

O modelo publish/subscribe também facilita o crescimento do projeto: hoje quem lê o tópico é o Node-RED e o Grafana; amanhã pode ser um módulo de IA ou um sistema de prontuário, sem mexer no código do ESP32.

---

## 3. Configuração da comunicação MQTT

**Broker.** Usei o broker público `broker.hivemq.com`, na porta `1883`. O enunciado cita o HiveMQ Cloud apenas como exemplo; o broker público é a escolha mais prática para uma simulação porque funciona direto no Wokwi e no Node-RED, sem precisar de conta, senha ou certificado. Em produção, a recomendação seria exatamente o HiveMQ Cloud com usuário, senha e TLS na porta 8883 — volto a isso na seção de segurança.

**Tópicos.** Organizei a comunicação em dois tópicos, com um sufixo próprio para não se misturar com dados de outras pessoas no broker público:

| Tópico | Para que serve |
|--------|----------------|
| `cardioia/leticia567501/telemetria` | Cada leitura dos sensores, em JSON. É o que alimenta o dashboard. |
| `cardioia/leticia567501/alerta` | Publicado (com flag *retida*) quando há alerta clínico ou aviso de sistema. Útil para o Grafana e para futuras automações. |

A mensagem de alerta é publicada como **retida** (*retained*): o broker guarda a última, então quem abrir o painel depois já vê o estado atual sem esperar a próxima leitura.

**Formato da mensagem (payload).** Cada leitura é publicada como um JSON de campos simples, fácil de ler em qualquer ferramenta:

```json
{"seq":42,"t":215,"temp":38.4,"umid":55.0,"bpm":128,
 "status":"FEBRE|TAQUICARDIA","alerta":1,"febre":1,"taqui":1,"origem":"live"}
```

O campo `alerta` (0/1) já diz se a leitura é crítica, o que deixa o dashboard simples: não precisa recalcular regra nenhuma, só reagir a esse campo. O campo `origem` distingue uma leitura enviada ao vivo (`live`) de uma que estava guardada e foi reenviada depois (`buffer`).

**Qualidade de serviço (QoS).** A publicação usa QoS 0 ("no máximo uma vez"), que é o suficiente para telemetria contínua: se uma leitura a cada três segundos se perder, a próxima chega logo. A resiliência de verdade (não perder dados numa queda) é resolvida no próprio ESP32, como explico a seguir, e não depende do QoS.

---

## 4. Resiliência: o que acontece quando a conexão cai

A chave deslizante do circuito simula a conectividade do paciente (às vezes o Wi-Fi de casa cai, o celular fica sem bateria, a pessoa viaja). O comportamento foi pensado para **não perder leitura nenhuma**:

- Enquanto o paciente está "online" e o broker está conectado, cada leitura é publicada na hora e o LED verde fica aceso.
- Quando fica "offline", as leituras **não são descartadas**: entram numa fila em memória (store-and-forward, o mesmo padrão da Parte 1). O dispositivo continua lendo e classificando normalmente, e o LED vermelho segue acendendo nos alertas.
- Quando a conexão volta, a fila é **esvaziada primeiro** — as leituras pendentes são reenviadas, das mais antigas para as mais novas — e só então o fluxo ao vivo continua. As reenviadas vão marcadas com `origem: buffer`.
- A fila tem tamanho limitado (300 leituras ≈ 15 minutos). Se encher, as mais antigas cedem lugar e o sistema avisa quantas foram descartadas. Esse limite curto é proposital: a Parte 1 já cuida do armazenamento longo (12 horas em SPIFFS); aqui a fila cobre as quedas rápidas de rede.

---

## 5. Configuração do dashboard no Node-RED

O dashboard foi montado no Node-RED com a paleta **node-red-dashboard** e está pronto no arquivo `flows.json` (basta importar). O fluxo tem a seguinte lógica:

1. Um nó **mqtt in** se inscreve no tópico de telemetria e já entrega a mensagem como objeto JSON.
2. A partir dele, a mensagem segue em paralelo para alguns nós **change**, que extraem o campo certo para cada widget (o `bpm` para o gráfico e o gauge de BPM, a `temp` para o gauge de temperatura, o `status` para o texto).
3. A mensagem completa também vai direto para um nó **ui_template**, que é o indicador de alerta.

Os widgets exibidos são:

| Widget | O que mostra | Observação |
|--------|--------------|------------|
| Gráfico de linha | Batimentos (BPM) ao longo do tempo | O "sinal vital em gráfico" que o enunciado pede |
| Medidor (gauge) | Temperatura, de 30 a 42 °C | Fica amarelo a partir de 37,5 °C e vermelho a partir de 38 °C |
| Medidor (gauge) | Frequência cardíaca | Fica vermelho acima de 120 bpm |
| Indicador de alerta | Faixa que fica **verde (Normal)** ou **vermelha (ALERTA)** | Reage ao campo `alerta` e mostra qual é o alerta |
| Texto | Status atual (NORMAL, FEBRE, TAQUICARDIA...) | |

**Limites de alerta** (os mesmos definidos pelo grupo na Parte 1): temperatura ≥ 38 °C (febre), frequência acima de 120 bpm (taquicardia). Quando a leitura cruza qualquer um deles, o campo `alerta` vem como 1, a faixa fica vermelha, os medidores entram na zona vermelha e o LED vermelho do ESP32 acende — tudo ao mesmo tempo.

---

## 6. Visualização avançada com Grafana (opcional)

Para a visualização avançada, o Grafana Cloud consome os mesmos dados sem exigir mudança no ESP32. O caminho usado é: um **plugin de MQTT** no Grafana se inscreve no tópico de telemetria, ou, de forma ainda mais robusta, o Node-RED grava cada leitura num banco de séries temporais (InfluxDB) que o Grafana lê. A partir daí dá para montar painéis com histórico mais longo, réguas de limite e alertas por e-mail. O passo a passo detalhado está no `GUIA-EXECUCAO.md`.

---

## 7. Segurança e boas práticas em IoT médico

Como são dados de saúde, mesmo num protótipo vale registrar o que mudaria num sistema real:

- **Broker fechado.** O broker público é aberto — qualquer um que saiba o tópico pode ler. Em produção, usuário, senha e **TLS** (porta 8883) são obrigatórios, e o tópico deveria ser imprevisível.
- **Sem dado pessoal no payload.** A mensagem só leva o número da leitura e os valores, nenhum nome ou documento do paciente. A identificação ficaria num cadastro separado e protegido, o que ajuda a atender a LGPD.
- **Decisão crítica na borda.** O alerta não espera a nuvem: o ESP32 acende o LED sozinho. Assim, se a internet cair, o cuidado imediato não para.
- **Resiliência.** Nenhuma leitura é perdida numa queda de rede, o que num monitoramento cardíaco pode ser a diferença entre ver ou não um episódio de taquicardia.

---

## 8. Limitações

- No Wokwi a internet vem da rede simulada `Wokwi-GUEST`; num ESP32 real entraria o Wi-Fi da casa do paciente.
- O botão substitui um sensor de pulso real (como o MAX30102); a lógica de média dos intervalos é a mesma.
- O broker público pode ter instabilidade por ser compartilhado; para a entrega, o tópico tem um sufixo próprio para reduzir interferência.
- Sem relógio de tempo real, o campo `t` conta os segundos desde que o dispositivo ligou; em produção o horário viria de um servidor NTP.

---

## 9. Conclusão

A Parte 2 fecha o ciclo da fase: o que a Parte 1 coletava e guardava na borda agora **sobe para a nuvem por MQTT e vira informação visível e acionável** num dashboard, com medidores e alerta automático, mais a opção de visualização avançada no Grafana. O conjunto mostra, de ponta a ponta, como Edge, Fog e Cloud se encaixam num caso real de saúde digital, com atenção à eficiência (protocolo leve), à resiliência (nada se perde offline) e à segurança dos dados.
