# Incidente 3 — Relógio da NOC-CLIENT fora de sincronia

**Tipo:** Real, espontâneo (não simulado)
**Host afetado:** NOC-CLIENT
**Categoria:** Alerta de Monitoramento (Zabbix) / Disponibilidade
**Trigger:** `Linux: System time is out of sync (diff with Zabbix server > 60s)`
**Início:** 05:53:39 PM
**Resolução:** 06:18:40 PM
**Duração:** 25m 1s

## Sintoma

O Zabbix disparou um alerta de severidade Warning indicando que o relógio da NOC-CLIENT estava dessincronizado do relógio do Zabbix server em mais de 60 segundos.

## Impacto

Diferença de relógio entre host monitorado e servidor de monitoramento compromete a precisão de timestamps de eventos, dificultando correlação de logs e podendo mascarar ou distorcer a ordem real dos acontecimentos durante uma investigação de incidente.

## Contexto

Esse não foi um teste planejado — foi um efeito colateral recorrente do ambiente: as VMs do laboratório atrasam o relógio interno depois de pausas/reinícios do host físico (o mesmo comportamento de base que já havia causado o Incidente 1, de corrupção de permissões no MariaDB da NOC-SERVER). Dessa vez, o desvio foi grande o suficiente para o próprio template "Linux by Zabbix agent" da NOC-CLIENT detectar e alertar automaticamente, funcionando exatamente como a melhoria de monitoramento sugerida na prevenção do Incidente 1.

## Diagnóstico

1. Alerta recebido no Zabbix: diferença de relógio > 60s entre NOC-CLIENT e o servidor
2. Verificação do relógio da NOC-CLIENT com:
   ```bash
   timedatectl
   ```
   Confirmado que o relógio do sistema estava atrasado em relação ao horário real, e que a sincronização automática (NTP) não estava corrigindo o desvio a tempo.

## Ação (primeiro combate N1)

1. Desativada a sincronização NTP automática temporariamente:
   ```bash
   timedatectl set-ntp false
   ```
2. Horário ajustado manualmente para o valor correto:
   ```bash
   timedatectl set-time "AAAA-MM-DD HH:MM:SS"
   ```
3. Sincronização NTP reativada após o ajuste, para manter o relógio correto dali em diante:
   ```bash
   timedatectl set-ntp true
   ```

## Resultado

O alerta fechou sozinho no Zabbix (transição automática Problem → Resolved) assim que o relógio da NOC-CLIENT voltou a ficar dentro da tolerância de 60s em relação ao Zabbix server, após 25 minutos e 1 segundo desde o início do problema.

## Evidências

Os prints de terminal desse incidente específico não foram preservados no momento em que ele ocorreu. A documentação foi reconstruída a partir das evidências nativas do próprio Zabbix:

- Registro do evento na tela de **Problems** do Zabbix, com host, severidade, horário de início e horário de resolução (ver `evidencias/Kali_Zabbix_Problems.png`)
- Duração exata do problema calculada automaticamente pelo Zabbix (25m 1s), confirmando o tempo entre a detecção do desvio e a correção manual

## Prevenção

- Configurar o serviço de sincronização de horário (`systemd-timesyncd` ou `chrony`) para reagir mais rápido a desvios grandes, em vez de depender só do ajuste periódico padrão
- Adicionar ao checklist de rotina do NOC: sempre validar `timedatectl` em todas as VMs logo após retomar o ambiente de uma pausa prolongada do host
- Manter prints/logs de terminal salvos automaticamente durante testes e incidentes reais (ex: `script` ou redirecionamento de output para arquivo), para que evidências não se percam caso o incidente ocorra fora de uma sessão de trabalho monitorada
