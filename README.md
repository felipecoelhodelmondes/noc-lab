# NOC Lab — Laboratório de Monitoramento (Zabbix + Grafana + GLPI)

Laboratório de infraestrutura real (não simulado), montado em máquinas virtuais, reproduzindo o dia a dia de um **analista de NOC N1**: monitorar disponibilidade e performance, diagnosticar e conter incidentes, registrar evidências técnicas e escalonar via ITSM.

Este projeto é a evolução de um projeto anterior de SOC (simulador de incidentes por IA), seguindo a mesma metodologia: **monta → configura → incidente → investiga → resolve → documenta**.

## Por que infraestrutura real em vez de simulador?

A decisão foi usar VMs reais no VirtualBox em vez de containers Docker, para ficar mais fiel a um ambiente de NOC de verdade — com todos os problemas reais que isso traz (relógio dessincronizado corrompendo banco, pacotes desatualizados, incompatibilidades de versão), que acabaram virando os melhores incidentes documentados.

## Arquitetura

```
                      ┌─────────────────────────┐
                      │   Rede Interna NOC-LAB  │
                      │   (VirtualBox Internal) │
                      └─────────────────────────┘
                       │           │          │
              ┌────────┘           │          └─────────────┐
              │                    │                        │
     ┌────────▼───────────┐ ┌──────▼─────────────┐ ┌────────▼─────────┐
     │   NOC-SERVER       │ │   NOC-CLIENT       │ │      KALI        │
     │ 192.168.100.10     │ │ 192.168.100.20     │ │ 192.168.100.30   │
     │ Ubuntu Server 26.04│ │ Ubuntu Server 26.04│ │   Kali Linux     │
     │                    │ │                    │ │                  │
     │ • Zabbix Server 7.0│ │ • Zabbix Agent2    │ │ • nmap, dig, curl│
     │ • Zabbix Frontend  │ │                    │ │   (testes        │
     │ • GLPI 11.0.8      │ │                    │ │   controlados)   │
     │ • Grafana          │ │                    │ │                  │
     └────────────────────┘ └────────────────────┘ └──────────────────┘
```

Cada VM tem dois adaptadores de rede: **NAT** (acesso à internet, para atualizações) e **Rede Interna "NOC-LAB"** (IP fixo, isolada, onde a monitoração acontece).

Veja [evidencias/VirtualBox.png](evidencias/VirtualBox.png) e [evidencias/NOC-Server_IPa.png](evidencias/NOC-Server_IPa.png) para a configuração de rede real.

## Stack

| Componente | Versão | Papel |
|---|---|---|
| Zabbix Server + Frontend | 7.0 LTS | Coleta e avaliação de métricas, disparo de alertas |
| Zabbix Agent2 | 7.0 LTS | Agente de coleta na NOC-CLIENT |
| Grafana | mais recente | Visualização — dashboards consolidados |
| GLPI | 11.0.8 | ITSM — abertura, ciclo de vida e fechamento de chamados |

## Fluxo operacional (o que este lab reproduz)

```
Zabbix detecta um problema
        │
        ▼
Grafana mostra o impacto no dashboard
        │
        ▼
Webhook Zabbix → GLPI abre chamado automaticamente (grupo N1)
        │
        ▼
Diagnóstico inicial e primeiro combate (N1)
        │
        ▼
Evidências registradas no chamado
        │
        ├── Resolvido pelo N1 → documenta e encerra
        │
        └── Não resolvido → escalona para N2/N3 (ver escalonamento.md)
```

## GLPI — estrutura ITSM

- **Grupos**: N1 - Monitoramento, N2 - Infraestrutura, N3 - Especialistas
- **Categorias ITIL**: Disponibilidade, Performance, Alerta de Monitoramento (Zabbix), Rede/Conectividade

## Integração Zabbix → GLPI

Chamados são abertos automaticamente no GLPI via webhook oficial do Zabbix (`media_glpi.yaml`), usando a API v2/OAuth2 do GLPI. Configuração:

- Cliente OAuth dedicado no GLPI
- Perfil de permissão dedicado (Chamados: Update/Create/See all; Followups: Add)
- Usuário técnico `zabbix-webhook`
- Macro global `{$ZABBIX.URL}` no Zabbix
- Media type importado do branch `release/7.0` do repositório oficial (o branch `master` causa erro de versão incompatível)
- Action de trigger disparando para severidade ≥ Warning

Essa integração já está validada em produção no lab — ver [evidencias/Kali_Zabbix_ActionLog.png](evidencias/Kali_Zabbix_ActionLog.png) (status "Sent") e [evidencias/Kali_GLPI_Chamado.png](evidencias/Kali_GLPI_Chamado.png) (chamado aberto e resolvido automaticamente, com todos os dados do problema do Zabbix).

## Dashboards

Grafana conectado ao Zabbix via plugin `alexanderzobnin-zabbix-datasource`, autenticado por API token de um usuário dedicado (`grafana-api`) com permissão **somente leitura** — por segurança, o Grafana nunca tem acesso de escrita ao Zabbix.

Dashboard consolidado **NOC-CLIENT** com 6 painéis: CPU, Memory %, Disk, Network Traffic, Uptime, FreeSwap %.

Ver [evidencias/Kali_Grafana_Dashboard.png](evidencias/Kali_Grafana_Dashboard.png).

## Incidentes documentados

| # | Incidente | Tipo | Runbook |
|---|---|---|---|
| 1 | Dados travados no Zabbix/Grafana por relógio dessincronizado (corrompeu permissões do MariaDB) | Real, espontâneo | [incidentes/falha-relogio-mariadb.md](incidentes/falha-relogio-mariadb.md) |
| 2 | Disco cheio na NOC-CLIENT (~97% de uso) | Simulado | [incidentes/incidente-disco-cheio.md](incidentes/incidente-disco-cheio.md) |
| 3 | Relógio da NOC-CLIENT fora de sincronia (`System time is out of sync`) | Real, espontâneo | [incidentes/incidente-relogio-sync.md](incidentes/incidente-relogio-sync.md) |
| 4 | CPU acima de 90% por 5 minutos na NOC-CLIENT | Simulado | [incidentes/incidente-cpu-alta.md](incidentes/incidente-cpu-alta.md) |

Todos seguem o mesmo formato: **Sintoma → Impacto → Evidências → Diagnóstico → Ação → Resultado → Prevenção**.

## Escalonamento

A matriz completa de escalonamento N1 → N2 → N3 está em [escalonamento.md](escalonamento.md).

## Evidências

Prints reais de dashboards, logs e chamados estão em [`evidencias/`](evidencias/):

- [VirtualBox.png](evidencias/VirtualBox.png) — as 3 VMs configuradas
- [NOC-Server_IPa.png](evidencias/NOC-Server_IPa.png) — rede da NOC-SERVER (`ip a`)
- [Kali_Zabbix_Dashboard.png](evidencias/Kali_Zabbix_Dashboard.png) — Global view do Zabbix
- [Kali_Zabbix_LatestData.png](evidencias/Kali_Zabbix_LatestData.png) — dados coletados em tempo real (77 itens da NOC-CLIENT)
- [Kali_Zabbix_Problems.png](evidencias/Kali_Zabbix_Problems.png) — histórico de problemas/alertas
- [Kali_Zabbix_ActionLog.png](evidencias/Kali_Zabbix_ActionLog.png) — webhook GLPI disparando com sucesso
- [Kali_GLPI_Chamado.png](evidencias/Kali_GLPI_Chamado.png) — chamado aberto e resolvido automaticamente no GLPI
- [Kali_Grafana_Dashboard.png](evidencias/Kali_Grafana_Dashboard.png) — dashboard consolidado NOC-CLIENT

## Baseado em

Estrutura de README inspirada no repositório [rafaellima418/homelab-rede](https://github.com/rafaellima418/homelab-rede).

## Projeto anterior

Este lab é a continuação de um projeto de SOC (simulador de incidentes por IA), aplicando a mesma metodologia a um cenário de NOC com infraestrutura real.
