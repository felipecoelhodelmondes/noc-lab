# Matriz de Escalonamento

Este documento define como um chamado aberto pelo Zabbix/GLPI é tratado e, quando necessário, escalonado dentro da estrutura do NOC.

## Níveis

| Nível | Grupo GLPI | Responsabilidade |
|---|---|---|
| N1 | N1 - Monitoramento | Primeiro contato: reconhece o alerta, faz diagnóstico inicial, aplica primeiro combate (ações básicas e reversíveis), registra evidências |
| N2 | N2 - Infraestrutura | Problemas que exigem acesso mais profundo à infraestrutura (rede, sistema operacional, banco de dados) ou que o primeiro combate do N1 não resolveu |
| N3 | N3 - Especialistas | Problemas complexos, recorrentes ou que exigem mudança estrutural/arquitetural |

## Critérios de escalonamento N1 → N2

Escalona para N2 quando:
- O primeiro combate do N1 não resolveu o problema dentro do tempo esperado
- O sintoma indica causa fora do escopo de monitoramento (ex: falha de hardware, corrupção de banco de dados, problema de configuração de rede)
- A ação necessária exige privilégios ou acesso que o N1 não possui
- Há suspeita de impacto em múltiplos serviços/hosts simultaneamente

## Critérios de escalonamento N2 → N3

Escalona para N3 quando:
- O problema é recorrente (já ocorreu antes e voltou a acontecer)
- Exige mudança de arquitetura, configuração estrutural ou decisão que impacta todo o ambiente
- Envolve investigação aprofundada de causa raiz que o N2 não conseguiu concluir

## Categorias ITIL usadas

| Categoria | Quando usar |
|---|---|
| Disponibilidade | Host/serviço fora do ar, timeout, restart inesperado |
| Performance | CPU, memória, disco, latência acima do limite aceitável |
| Alerta de Monitoramento (Zabbix) | Falhas do próprio agente/servidor de monitoramento |
| Rede/Conectividade | Problemas de rede, DNS, roteamento, firewall |

## Severidades (Zabbix) x Prioridade (GLPI)

| Severidade Zabbix | Prioridade GLPI sugerida |
|---|---|
| Disaster | Muito alta |
| High | Alta |
| Average | Média |
| Warning | Baixa/Média |
| Information | Baixa |

## Fluxo resumido

1. Zabbix dispara alerta (severidade ≥ Warning)
2. Webhook abre chamado automaticamente no GLPI, grupo **N1 - Monitoramento**
3. N1 registra diagnóstico inicial e evidências no chamado
4. Se resolvido → documenta e encerra
5. Se não resolvido dentro do prazo/escopo → escalona para **N2 - Infraestrutura**, anexando tudo que já foi levantado
6. Se N2 não resolver → escalona para **N3 - Especialistas**
7. Ao encerrar, o chamado deve conter: sintoma, evidências coletadas, diagnóstico, ação tomada, resultado e recomendação de prevenção

## Exemplo real do fluxo funcionando

O incidente de CPU alta (`incidentes/incidente-cpu-alta.md`) demonstra o fluxo completo: alerta do Zabbix → webhook dispara → chamado aberto automaticamente no GLPI com todos os dados do problema → resolvido e fechado com os dois eventos (abertura e resolução) documentados no mesmo chamado. Ver `evidencias/Kali_Zabbix_ActionLog.png` e `evidencias/Kali_GLPI_Chamado.png`.
