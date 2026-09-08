# Incidente 4 — CPU alta na NOC-CLIENT (com abertura automática de chamado no GLPI)

**Tipo:** Simulado (controlado)
**Host afetado:** NOC-CLIENT
**Categoria:** Performance
**Trigger:** `Linux: High CPU utilization (over 90% for 5m)`
**Início:** 14:30:34 (2026-09-08)
**Resolução:** 14:39:30 (2026-09-08) — 5m 0s de duração do problema

Este é o incidente que valida o fluxo completo do laboratório de ponta a ponta: **Zabbix detecta → webhook dispara → GLPI abre chamado automaticamente → resolução documentada no próprio chamado.**

## Sintoma

Utilização de CPU do NOC-CLIENT acima de 90% sustentada por mais de 5 minutos, disparando trigger de severidade Warning no Zabbix.

## Impacto

CPU saturada compromete o tempo de resposta de todos os serviços rodando no host, podendo causar timeouts em cima de outras verificações de monitoramento e degradar a experiência de qualquer aplicação hospedada ali.

## Reprodução (simulação controlada)

Carga de CPU gerada propositalmente com a ferramenta:

```bash
yes > /dev/null &
```

## Diagnóstico

1. Zabbix detectou utilização de CPU em **100%** (dado operacional confirmado no próprio alerta) e manteve o problema ativo por mais de 5 minutos, cumprindo a condição da trigger
2. Painel **CPU** do dashboard Grafana (NOC-CLIENT) capturou visualmente o pico sustentado
3. Webhook do Zabbix (integração com GLPI) disparou automaticamente a Action configurada para severidade ≥ Warning

## Ação (primeiro combate N1)

1. Confirmado que a carga era do teste controlado (processo `yes`)
2. Processo finalizado para liberar a CPU:
   ```bash
   pkill yes
   ```
3. Confirmado o retorno da utilização de CPU a níveis normais (~5% após a finalização, conforme registrado no chamado de resolução)

## Resultado — fluxo de integração comprovado

- **Zabbix → GLPI (abertura):** Action log do Zabbix registrou o envio com status **Sent**, mídia GLPI, mensagem contendo host, severidade, utilização de CPU e ID do problema original ([evidencias/Kali_Zabbix_ActionLog.png](evidencias/Kali_Zabbix_ActionLog.png))
- **Chamado criado automaticamente no GLPI** pelo usuário técnico `zabbix-webhook`, com todos os dados do problema preenchidos automaticamente: host NOC-CLIENT, severidade Warning, utilização de 100%, link direto de volta para o problema no Zabbix
- **Resolução também registrada automaticamente**: quando o alerta foi resolvido no Zabbix, o mesmo chamado recebeu uma atualização (followup) com a confirmação da resolução, horário e utilização final de CPU (5.06%)
- Chamado fechado com status **Solucionado** no GLPI ([evidencias/Kali_GLPI_Chamado.png](evidencias/Kali_GLPI_Chamado.png))
  
## Prevenção

- Investigar, em um cenário real (não simulado), quais processos costumam gerar picos de CPU recorrentes e considerar alertas de tendência, não só de threshold pontual
- Manter o webhook GLPI monitorado por um item de "meta-monitoramento" (alertar se o Action log começar a falhar), já que essa integração é crítica para o fluxo de ITSM funcionar
- Usar este incidente como referência/teste de fumaça sempre que a integração Zabbix↔GLPI for alterada, já que ele exercita o ciclo completo (abertura automática + atualização de resolução)
