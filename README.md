# VeloCloud SD-WAN Edge by HTTP (versão estendida)

Template Zabbix para monitorar **Edges VeloCloud SD-WAN (Arista, antes VMware)** pela API REST v1 do
Orchestrator (`/portal/rest`). Esta é uma versão revisada e ampliada do template oficial da Zabbix,
adaptada para monitorar a operação dos Edges no dia a dia: estado do Edge e dos links WAN em tempo real,
qualidade dos links, peers SD-WAN, QoE, prioridade de negócio e aplicações.

| | |
|---|---|
| **Arquivo** | [`template_velocloud_sd-wan_edge_http.yaml`](template_velocloud_sd-wan_edge_http.yaml) |
| **Zabbix** | 7.0 ou superior (formato de exportação `7.0`) |
| **Versão do template** | `7.0-2` |
| **Autor desta versão** | Mauro Paiva |
| **Grupos** | `Templates/Network devices`, `Personalizados` |
| **Tags** | `class: network`, `target: velocloud-sd-wan-edge` |

---

## Créditos e inspiração

Este trabalho deriva do template oficial **VeloCloud SD-WAN Edge by HTTP**, criado e mantido pela
**Zabbix** e distribuído junto com o produto. O crédito pela ideia, pela estrutura e pelos nomes de chaves
originais (`velocloud.edge.*`, `velocloud.link.*`, `velocloud.sdwan.*`) é da equipe de templates da Zabbix.

- Página da integração: <https://www.zabbix.com/br/integrations/velocloud>
- Código-fonte do template original (Zabbix 7.0):
  [`templates/net/velocloud_http`](https://github.com/zabbix/zabbix/tree/release/7.0/templates/net/velocloud_http)
- Recursos e documentação VeloCloud (Arista): <https://www.arista.com/en/support/velocloud-resources>
- Guia da API do Orchestrator (Portal API, VeloCloud SD-WAN 6.1, PDF):
  [VC-SD-WAN-6.1-Orchestrator-PortalAPI-Guide.pdf](https://www.arista.com/assets/data/pdf/qsg/velo-api/VC-SD-WAN-6.1-Orchestrator-PortalAPI-Guide.pdf)

O template original é distribuído com o Zabbix, que a partir da versão 7.0 é licenciado sob a
[GNU AGPL v3](https://www.gnu.org/licenses/agpl-3.0.html).

---

## Sumário

- [O que mudou em relação ao template oficial](#o-que-mudou-em-relação-ao-template-oficial)
- [Requisitos](#requisitos)
- [Instalação](#instalação)
- [Como a coleta funciona](#como-a-coleta-funciona)
- [Macros](#macros)
- [Itens](#itens)
- [Regras de descoberta (LLD)](#regras-de-descoberta-lld)
- [Triggers](#triggers)
- [Gráficos e dashboard](#gráficos-e-dashboard)
- [Value maps](#value-maps)
- [Ajustes e solução de problemas](#ajustes-e-solução-de-problemas)

---

## O que mudou em relação ao template oficial

| Área | Template oficial (Zabbix 7.0) | Esta versão |
|---|---|---|
| Coleta | 3 itens HTTP agent + 1 script, todos a cada `{$VELOCLOUD.EDGE.FREQUENCY}` | 4 scripts com cliente de API comum, validação de parâmetros e erro tratado em item próprio para cada um |
| Estado do Edge e dos links | Mesmo intervalo das métricas (15m) | `edge/getEdge` com `recentLinks` em intervalo próprio, `{$VELOCLOUD.EDGE.STATE.FREQUENCY}` (5m) |
| Janela de métricas | Sem compensação de atraso | Janela termina em `now - {$VELOCLOUD.METRICS.LAG}` para não ler buckets de 5 min ainda incompletos |
| Links WAN | Descobertos a partir das métricas (links sem tráfego somem) | Descobertos a partir do `getEdge`; links sem tráfego continuam monitorados, o que permite alertar `DISCONNECTED` |
| Métricas de link | Bytes, pacotes, latência e perda | Throughput (bps), banda medida, utilização %, latência, jitter, perda, sinal (wireless), estado VPN, estado de backup |
| Resumo de links | — | Total, estáveis, instáveis, desconectados e disponibilidade % |
| Peers SD-WAN | Contagem de caminhos | Contagem de caminhos + QoE voz, vídeo e transacional por peer e menor QoE do Edge |
| Caminhos | Bytes, pacotes e perda | Latência, jitter, perda, throughput, QoE e status por caminho, opcional via `{$VELOCLOUD.PATH.METRICS.ENABLED}` |
| Filtro de peers | — | Por nome e tipo, aplicado dentro do script antes das chamadas por caminho (economiza requisições) |
| Prioridade de negócio | — | Throughput por classe (High, Normal, Low, Control), recebido e enviado |
| Aplicações | — | Top N aplicações por volume (`getEdgeAppMetrics`), para habilitar só onde for necessário |
| Sistema | CPU, memória, flows, túneis, handoff | Inclui temperatura da CPU, valores máximos, uptime de serviço, último contato, build, serial, estado de serviço |
| Triggers | 7 (6 + 1 protótipo) | 31 (22 + 9 protótipos), com dependência no Edge OFFLINE, limites críticos e de alerta e macros com contexto por link e peer |
| Limiares | Fixos no template, janela de 15m | Macros de alerta e crítico, janelas configuráveis, contexto pelo nome do link (`{$VELOCLOUD.LINK.LATENCY.WARN:"NOME_DO_LINK"}`) |
| Itens brutos | Histórico fixo | Retenção controlada por `{$VELOCLOUD.RAW.HISTORY}` (1h em homologação, `0` em produção) |
| Dashboard | 4 páginas | 6 páginas em português (Visão geral, Links, Prioridade de negócio, Sistema, Peers SD-WAN, Aplicações) |
| Idioma | Inglês | Triggers, descrições, gráficos e dashboard em português; chaves e nomes de itens mantidos em inglês |
| Removido | Itens de inventário do site (cidade, país, contato, coordenadas) e `{$HTTP.TLS.VERIFY}` | — |

O nome e o UUID do template são os mesmos do oficial. Por isso esta versão continua funcionando com os
host prototypes do template **VeloCloud SD-WAN by HTTP** (Orchestrator), que criam os hosts dos Edges e já
preenchem `{$VELOCLOUD.EDGE.ID}` e `{$VELOCLOUD.ENTERPRISE.ID}`.

---

## Requisitos

- Zabbix server ou proxy **7.0+**, com acesso HTTPS ao Orchestrator (direto ou via proxy HTTP).
- Token de API do Orchestrator com permissão de leitura sobre os Edges e métricas da enterprise.
- ID da enterprise e ID de cada Edge no Orchestrator. Os dois aparecem na URL do portal ao abrir um Edge
  e também podem ser obtidos pela API (`enterprise/getEnterpriseEdges`). Os métodos usados pelo template
  estão descritos no [guia da API em PDF](https://www.arista.com/assets/data/pdf/qsg/velo-api/VC-SD-WAN-6.1-Orchestrator-PortalAPI-Guide.pdf).

---

## Instalação

1. **Importe o template.** Em *Coleta de dados → Templates → Importar*, selecione
   `template_velocloud_sd-wan_edge_http.yaml`.
   > Como o UUID é o mesmo do template oficial, a importação **substitui** o oficial, se ele existir.
   > Se marcar "Excluir ausentes", itens do oficial que não existem nesta versão serão removidos dos hosts.

2. **Crie um host por Edge** (ou use a descoberta do template **VeloCloud SD-WAN by HTTP**) e vincule o
   template. Interface não é necessária.

3. **Preencha as macros de conexão no host.** Elas existem no template, mas sem valor:

   | Macro | Tipo | Valor |
   |---|---|---|
   | `{$VELOCLOUD.URL}` | Texto | Endereço do Orchestrator, sem `https://`, ex. `vco.velocloud.net` |
   | `{$VELOCLOUD.TOKEN}` | Texto secreto | Token de API do Orchestrator |
   | `{$VELOCLOUD.ENTERPRISE.ID}` | Texto | ID da enterprise |
   | `{$VELOCLOUD.EDGE.ID}` | Texto | ID do Edge |

4. **Opcional: macros globais.** Para definir URL, token ou enterprise uma única vez em
   *Administração → Macros*, remova a macro correspondente do template. Uma macro vazia no template tem
   precedência sobre a global.

5. **Revise os filtros** de links e peers (veja [Macros](#macros)). Em Edges com muitos peers, restrinja
   `{$VELOCLOUD.LLD.PEER.NAME.MATCHES}` aos hubs e gateways que interessam, ex. `^(HUB-01|HUB-02)$`.

6. **Opcional:** habilite *Get edge application data* nos Edges em que a visibilidade por aplicação for
   necessária (hubs, sedes, centro de comando) e defina `{$VELOCLOUD.PATH.METRICS.ENABLED}=1` onde quiser
   métricas por caminho.

---

## Como a coleta funciona

Quatro itens do tipo **Script** chamam a API e devolvem um JSON já normalizado. Todos os outros itens são
**dependentes** desses quatro. Os scripts compartilham o mesmo cliente (`VC`), que valida os parâmetros
obrigatórios, envia `Authorization: Token <token>`, usa o proxy HTTP se configurado e trata erros HTTP, JSON
inválido e erros da API. Em caso de falha, o script devolve `{"error": "..."}` e o item
`... collection errors` correspondente dispara um trigger.

| Item (chave) | Chamadas na API | Intervalo | O que alimenta |
|---|---|---|---|
| Get edge data (`velocloud.edge.get.data`) | `edge/getEdge` com `recentLinks` | `{$VELOCLOUD.EDGE.STATE.FREQUENCY}` (5m) | Estado, ativação, serviço, HA, versões, uptime, último contato, resumo e descoberta dos links WAN |
| Get edge metrics (`velocloud.edge.get.metrics`) | `metrics/getEdgeStatusMetrics` e `metrics/getEdgeLinkMetrics` na mesma janela | `{$VELOCLOUD.EDGE.FREQUENCY}` (15m) | CPU, memória, temperatura, flows, túneis, handoff, throughput total e por prioridade, métricas por link |
| Get edge peer data (`velocloud.edge.get.peer.data`) | `edge/getEdgeSDWANPeers` e, se habilitado, `metrics/getEdgeSDWANPeerPathMetrics` por peer | `{$VELOCLOUD.EDGE.FREQUENCY}` (15m) | Peers, caminhos e QoE |
| Get edge application data (`velocloud.edge.get.flow.data`) | `metrics/getEdgeAppMetrics` (top N por `totalBytes`) | `{$VELOCLOUD.APP.FREQUENCY}` (1h) | Throughput e participação por aplicação |

**Janela de métricas.** A API agrega os dados em buckets de 5 minutos e pode levar até 10 minutos para
consolidá-los. Por isso a janela de cada coleta tem o tamanho do intervalo do item (mínimo de 15 minutos) e
termina em `agora - {$VELOCLOUD.METRICS.LAG}`. Bytes são convertidos em bps dividindo pelo tamanho da janela.

**Utilização do link** é o throughput dividido pela banda medida pelo Edge no melhor caminho
(`bpsOfBestPath*`), limitada a 100%.

---

## Macros

### Conexão e coleta

| Macro | Padrão | Descrição |
|---|---|---|
| `{$VELOCLOUD.URL}` | — | Endereço do Orchestrator, sem `https://`, ex. `vco.velocloud.net`. Definir no host. |
| `{$VELOCLOUD.TOKEN}` | — | Token de API (texto secreto). Definir no host. |
| `{$VELOCLOUD.ENTERPRISE.ID}` | — | ID da enterprise. Definir no host. |
| `{$VELOCLOUD.EDGE.ID}` | — | ID do Edge. Definir no host. |
| `{$VELOCLOUD.HTTP.PROXY}` | — | Proxy HTTP opcional, ex. `http://proxy:3128`. |
| `{$VELOCLOUD.EDGE.DATA.TIMEOUT}` | `30s` | Timeout dos scripts. Aumente se usar métricas por caminho em hubs com muitos peers. |
| `{$VELOCLOUD.EDGE.STATE.FREQUENCY}` | `5m` | Intervalo da coleta de estado do Edge e dos links. |
| `{$VELOCLOUD.EDGE.FREQUENCY}` | `15m` | Intervalo e janela das coletas de métricas e de peers. |
| `{$VELOCLOUD.APP.FREQUENCY}` | `1h` | Intervalo e janela da coleta de aplicações. |
| `{$VELOCLOUD.APP.TOP.N}` | `10` | Quantidade de aplicações coletadas por Edge. |
| `{$VELOCLOUD.METRICS.LAG}` | `10m` | Atraso aplicado ao fim da janela de métricas. |
| `{$VELOCLOUD.PATH.METRICS.ENABLED}` | `0` | `1` para coletar métricas por caminho (uma chamada extra por peer filtrado). |
| `{$VELOCLOUD.RAW.HISTORY}` | `1h` | Retenção dos itens brutos (JSON). Use `0` em produção. |
| `{$VELOCLOUD.NODATA.TIMEOUT}` | `30m` | Tempo sem dados da API até disparar o trigger de ausência de coleta. |

### Limiares

| Macro | Padrão | Descrição |
|---|---|---|
| `{$VELOCLOUD.EDGE.CPU.UTIL.WARN}` / `.CRIT` | `80` / `90` | Uso de CPU, em %. |
| `{$VELOCLOUD.EDGE.MEMORY.UTIL.WARN}` / `.CRIT` | `70` / `90` | Uso de memória, em %. |
| `{$VELOCLOUD.EDGE.TEMP.WARN}` / `.CRIT` | `75` / `85` | Temperatura da CPU, em °C. |
| `{$VELOCLOUD.EDGE.HANDOFF.DROPS.WARN}` | `0` | Descartes na fila de handoff por janela. |
| `{$VELOCLOUD.EDGE.UTIL.PERIOD}` | `30m` | Período avaliado pelos triggers de CPU, memória, temperatura e handoff. |
| `{$VELOCLOUD.EDGE.LASTCONTACT.MAX}` | `900` | Idade máxima do último heartbeat do Edge, em segundos. |
| `{$VELOCLOUD.LINK.LATENCY.WARN}` ¹ | `150` | Latência do link, em ms. |
| `{$VELOCLOUD.LINK.JITTER.WARN}` ¹ | `30` | Jitter do link, em ms. |
| `{$VELOCLOUD.LINK.LOSS.WARN}` ¹ | `2` | Perda de pacotes do link, em %. |
| `{$VELOCLOUD.LINK.UTIL.WARN}` ¹ | `90` | Utilização do link em relação à banda medida, em %. |
| `{$VELOCLOUD.LINK.FLAP.MAX}` ¹ | `4` | Mudanças de estado do link por hora. |
| `{$VELOCLOUD.LINK.PERIOD}` | `30m` | Período avaliado pelos triggers de qualidade de link. |
| `{$VELOCLOUD.PEER.QOE.WARN}` ² | `7` | QoE mínimo (0 a 10) dos peers. Reservada: nenhum trigger usa esta macro ainda. |

¹ Aceita contexto pelo nome do link, ex. `{$VELOCLOUD.LINK.LATENCY.WARN:"MEU_LINK_MPLS"}`.
² Aceita contexto pelo nome do peer.

### Filtros de descoberta

| Macro | Padrão | Descrição |
|---|---|---|
| `{$VELOCLOUD.LLD.LINKS.IF.FILTER.MATCHES}` | `^GE[0-9]+$` | Interfaces dos links WAN. Para incluir LTE e SFP: `^(GE\|SFP\|CELL)[0-9]+$`. |
| `{$VELOCLOUD.LLD.LINKS.NAME.FILTER.MATCHES}` | `.*` | Links a descobrir, pelo nome. |
| `{$VELOCLOUD.LLD.LINKS.NAME.FILTER.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Links a ignorar, pelo nome. |
| `{$VELOCLOUD.LLD.LINKS.WIRELESS.TYPE}` | `^(WIRELESS\|LTE\|CELLULAR\|5G)$` | `networkType` dos links que recebem o item *Signal strength*. |
| `{$VELOCLOUD.LLD.PEER.NAME.MATCHES}` | `.*` | Peers monitorados, pelo nome, ex. `^(HUB-01\|HUB-02)$`. Aplicado no script e no LLD. |
| `{$VELOCLOUD.LLD.PEER.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Peers a ignorar, pelo nome. |
| `{$VELOCLOUD.LLD.PEER.TYPE.MATCHES}` | `.*` | Tipos de peer (`HUB`, `GATEWAY`, `BRANCH`), ex. `^(HUB\|GATEWAY)$`. |
| `{$VELOCLOUD.LLD.APP.NAME.MATCHES}` | `.*` | Aplicações a descobrir. |
| `{$VELOCLOUD.LLD.APP.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Aplicações a ignorar. |

---

## Itens

Todos os itens abaixo, exceto os quatro scripts, são dependentes.

### Edge (`component: edge`)

| Nome | Chave | Origem |
|---|---|---|
| State | `velocloud.edge.state` | getEdge |
| Activation state | `velocloud.edge.activation` | getEdge |
| Service state | `velocloud.edge.service_state` | getEdge |
| HA state | `velocloud.edge.ha_state` | getEdge |
| Hostname (SD-WAN) | `velocloud.edge.hostname` | getEdge |
| Description | `velocloud.edge.description` | getEdge |
| Model number | `velocloud.edge.model` | getEdge |
| Serial number | `velocloud.edge.serial` | getEdge |
| Software version | `velocloud.edge.software_version` | getEdge |
| Build number | `velocloud.edge.build` | getEdge |

### Sistema, CPU e memória (`component: system`, `cpu`, `memory`)

| Nome | Chave | Origem |
|---|---|---|
| System uptime | `velocloud.edge.system_uptime` | getEdge |
| Service uptime | `velocloud.edge.service_uptime` | getEdge |
| Last contact age | `velocloud.edge.last_contact` | getEdge |
| CPU usage in percent / (max) | `velocloud.edge.cpu.usage` / `.max` | Métricas |
| CPU core temperature | `velocloud.edge.cpu.core.temp` | Métricas |
| Memory usage in percent / (max) | `velocloud.edge.memory.usage` / `.max` | Métricas |
| Flow count | `velocloud.edge.flow.count` | Métricas |
| Tunnel count / V6 | `velocloud.edge.tunnel.count` / `.v6` | Métricas |
| Handoff queue drops | `velocloud.edge.handoff.queue.drops` | Métricas |
| Get ... collection errors | `velocloud.edge.get.*.error` | Um por script |

### Links WAN e tráfego (`component: link`, `traffic`)

| Nome | Chave | Origem |
|---|---|---|
| WAN links: total / stable / unstable / disconnected | `velocloud.edge.links.{total,stable,unstable,disconnected}` | getEdge |
| WAN links: availability, % | `velocloud.edge.links.stable.pct` | getEdge |
| Throughput in / out (all links) | `velocloud.edge.throughput.{rx,tx}` | Métricas |
| Business priority High / Normal / Low / Control: throughput in / out | `velocloud.edge.priority.{high,normal,low,control}.{rx,tx}` | Métricas |

### QoE (`component: qoe`)

| Nome | Chave | Origem |
|---|---|---|
| QoE voice / video / transactional (min across peers) | `velocloud.edge.qoe.{voice,video,trans}` | Peers |

---

## Regras de descoberta (LLD)

### Link discovery (`velocloud.link.discovery`)

Fonte: *Get edge data*. Filtros por interface e nome. Links que deixam de aparecer nunca são desativados,
para que o estado `DISCONNECTED` continue sendo alertado. Um override só cria *Signal strength* em links cujo
`networkType` casa com `{$VELOCLOUD.LLD.LINKS.WIRELESS.TYPE}`.

Protótipos de item (`Link [{#NAME}]:[{#IP}]: ...`): State, VPN state, Backup state, Last active, Throughput
in/out, Bandwidth in/out (measured), Utilization in/out %, Best latency rx/tx, Best jitter rx/tx, Best loss
rx/tx, Signal strength, Raw data, Raw status.

### SD-WAN peer discovery (`velocloud.sdwan.peer.discovery`)

Fonte: *Get edge peer data*. Filtros por nome e tipo do peer.

Protótipos de item (`SD-WAN Peer [{#NAME}]:[{#TYPE}]: ...`): Description, Total / Stable / Unstable /
Standby / Dead / Unknown path, QoE voice / video / transactional, Raw data.

### SD-WAN peer path discovery (`velocloud.sdwan.path.discovery`)

Fonte: *Get edge peer data*, somente com `{$VELOCLOUD.PATH.METRICS.ENABLED}=1`. Caminhos não vistos por
1 dia são desativados.

Protótipos de item (`Path [{#SOURCE.LINK}] => [{#DESTINATION}:{#DESTINATION.LINK}]: ...`): Status,
Latency rx/tx, Jitter rx/tx, Packet loss rx/tx, Throughput rx/tx, QoE voice, Raw data.

### Application discovery (`velocloud.app.discovery`)

Fonte: *Get edge application data*. Filtro por nome da aplicação. Aplicações fora do top N por 1 dia são
desativadas.

Protótipos de item (`App [{#APP.NAME}]: ...`): Throughput in/out, Traffic share, Raw data.

---

## Triggers

Os triggers de recurso, links e túneis dependem de **Edge está "OFFLINE"**, evitando cascata de alertas
quando o Edge cai. Os nomes seguem o padrão `VeloCloud Edge: [{HOST.NAME}] ...`.

### Edge

| Severidade | Nome | Condição |
|---|---|---|
| Disaster | Edge está "OFFLINE" | `state = OFFLINE` |
| High | Edge conectado sem túneis SD-WAN | Edge `CONNECTED` e `tunnel.count = 0` |
| High | Estado HA em "FALHA" | `ha_state = FAILED` |
| High | Uso de CPU crítico | CPU > `CRIT` durante `{$VELOCLOUD.EDGE.UTIL.PERIOD}` |
| High | Uso de memória crítico | Memória > `CRIT` durante o período |
| High | Temperatura da CPU crítica | Temperatura > `CRIT` durante o período |
| Average | Edge em estado "DEGRADED" | `state = DEGRADED` |
| Average | Alto uso de CPU | CPU > `WARN` durante o período |
| Average | Alto uso de memória | Memória > `WARN` durante o período |
| Average | Sem dados da API do Orchestrator | `nodata` por `{$VELOCLOUD.NODATA.TIMEOUT}` |
| Warning | Edge fora de serviço (OUT_OF_SERVICE) | `service_state = OUT_OF_SERVICE` |
| Warning | Edge sem contato com o Orchestrator | Último contato > `{$VELOCLOUD.EDGE.LASTCONTACT.MAX}` |
| Warning | Perda de redundância de links WAN | 2 ou mais links e no máximo 1 estável |
| Warning | Temperatura da CPU elevada | Temperatura > `WARN` durante o período |
| Warning | Descartes na fila de handoff | Drops > limite durante o período (**desativado por padrão**) |
| Info | Edge reiniciado | System uptime < 10 min |
| Info | Serviço do Edge reiniciado | Service uptime < 10 min sem reboot do equipamento |
| Info | Versão de software alterada | Mudança em `software_version` |

### Coleta

| Severidade | Nome |
|---|---|
| High | Falha ao coletar dados de métricas (*Get edge data*) |
| Average | Falha ao coletar métricas de saúde e links |
| Warning | Falha ao coletar dados de peers SD-WAN |
| Info | Falha ao coletar dados de aplicações |

### Links (protótipos)

| Severidade | Nome | Condição |
|---|---|---|
| High | Link Indisponível | Estado `DISCONNECTED` |
| Average | Overlay VPN indisponível com link ativo | VPN indisponível com link `STABLE` |
| Average | Perda de pacotes alta | Perda rx ou tx > limite durante `{$VELOCLOUD.LINK.PERIOD}` |
| Warning | Link instável | Estado `UNSTABLE` |
| Warning | Link oscilando (flapping) | Mudanças de estado em 1h > `{$VELOCLOUD.LINK.FLAP.MAX}` |
| Warning | Latência alta | Latência rx ou tx > limite durante o período |
| Warning | Jitter alto | Jitter rx ou tx > limite durante o período |
| Warning | Utilização alta | Utilização rx ou tx > limite durante o período |

### Peers e caminhos (protótipos)

| Severidade | Nome | Condição |
|---|---|---|
| Warning | Caminho DEAD | Status do caminho `DEAD` |

Os limites de link aceitam macros com contexto pelo nome do link, para tratar links específicos de forma
diferente sem precisar de outro template.

---

## Gráficos e dashboard

**Gráficos do template:** CPU e memória, Flows e túneis, Links WAN, Prioridade de negócio (enviado e
recebido), QoE (menor entre peers), Temperatura da CPU e Throughput total.

**Protótipos de gráfico:** por link (Best latency, Best jitter, Best loss, Throughput x banda medida,
Utilização), por peer (Path overview, QoE), por caminho (Latency e loss) e por aplicação (Throughput).

**Dashboard `VeloCloud: General`**, com 6 páginas:

| Página | Conteúdo |
|---|---|
| Visão geral | Estado, ativação, serviço, HA, modelo, versão, uptime e último contato; gauges de CPU, memória, temperatura e QoE; honeycomb de estado dos links; throughput total; distribuição de tráfego por link e por prioridade |
| Links | Throughput, utilização, banda medida, latência, jitter, perda e estado por link |
| Prioridade de negócio | Throughput e participação por classe, recebido e enviado |
| Sistema | CPU, memória, temperatura, flows, túneis e drops de handoff |
| Peers SD-WAN | Caminhos estáveis, DEAD e total por peer; QoE voz, vídeo e transacional por peer |
| Aplicações | Top aplicações recebido, enviado e participação no volume |

---

## Value maps

| Nome | Valores |
|---|---|
| Edge states | 0 OFFLINE, 1 CONNECTED, 2 NEVER_ACTIVATED, 3 DEGRADED, 10 UNKNOWN |
| Edge activation state | 0 PENDING, 1 ACTIVATED, 2 UNASSIGNED, 3 REACTIVATION_PENDING, 10 UNKNOWN |
| Service state | 0 OUT_OF_SERVICE, 1 IN_SERVICE, 2 PENDING_SERVICE, 10 UNKNOWN |
| Edge HA status | 0 UNCONFIGURED, 1 READY, 2 PENDING_INIT, 3 FAILED, 10 UNKNOWN |
| Link states | 0 UNSTABLE, 1 STABLE, 2 DISCONNECTED, 3 QUIET, 4 INITIAL, 10 UNKNOWN |
| Link backup state | 0 UNCONFIGURED, 1 STANDBY, 2 ACTIVE, 10 UNKNOWN |
| Path states | 0 UNSTABLE, 1 STABLE, 2 DEAD, 3 STANDBY, 10 UNKNOWN |

---

## Ajustes e solução de problemas

- **Rate limit.** Com os valores padrão, cada Edge faz 1 chamada a cada 5 minutos (`getEdge`), 3 a cada
  15 minutos (status, links e peers) e, se habilitada, 1 por hora (aplicações). Com métricas por caminho
  ativas, soma-se uma chamada por peer filtrado a cada 15 minutos. Consulte a política de rate limit do
  Orchestrator antes de reduzir os intervalos ou ativar caminhos em muitos Edges.
- **Inspecionar o JSON.** Com `{$VELOCLOUD.RAW.HISTORY}=1h`, o JSON de cada script fica visível em
  *Dados mais recentes*. Em produção use `0` para não gravar JSON no banco.
- **Erros de coleta.** Os itens `velocloud.edge.get.*.error` mostram a mensagem retornada (HTTP, JSON
  inválido, erro da API ou macro não definida). Com `DebugLevel=4` o server/proxy registra o status HTTP de
  cada chamada no log, com o prefixo `[ VeloCloud ]`.
- **"Parametro obrigatorio nao definido".** Alguma das macros `{$VELOCLOUD.URL}`, `{$VELOCLOUD.TOKEN}`,
  `{$VELOCLOUD.ENTERPRISE.ID}` ou `{$VELOCLOUD.EDGE.ID}` não foi resolvida para o host.
- **Timeout com métricas por caminho.** Hubs com muitos peers fazem muitas chamadas na mesma execução.
  Restrinja os peers pelos filtros ou aumente `{$VELOCLOUD.EDGE.DATA.TIMEOUT}`.
- **Links LTE/SFP não aparecem.** Ajuste `{$VELOCLOUD.LLD.LINKS.IF.FILTER.MATCHES}` para
  `^(GE|SFP|CELL)[0-9]+$`.
- **Peers não aparecem.** Verifique `{$VELOCLOUD.LLD.PEER.NAME.MATCHES}`: o filtro é aplicado no script,
  então peers fora dele não chegam nem ao JSON bruto.
