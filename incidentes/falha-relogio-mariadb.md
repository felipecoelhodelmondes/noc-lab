# Incidente 1 — Dados travados no Zabbix/Grafana (relógio dessincronizado corrompeu MariaDB)

**Tipo:** Real, espontâneo (não simulado)
**Host afetado:** NOC-SERVER
**Categoria:** Disponibilidade

## Sintoma

Zabbix e Grafana pararam de mostrar dados atualizados — os dashboards ficaram "travados", sem novos valores chegando, mesmo com o Zabbix Agent2 da NOC-CLIENT aparentemente ativo.

## Impacto

Perda de visibilidade em tempo real de todo o ambiente monitorado. Sem histórico atualizado, qualquer incidente real que ocorresse nesse período não seria detectado nem alertado.

## Contexto

As VMs do laboratório sofrem desync recorrente de relógio: depois de pausar ou reiniciar o host físico, o relógio interno das VMs pode atrasar horas ou até dias. Esse comportamento já havia causado erros de "not valid yet" no `apt` anteriormente, sempre corrigidos ajustando a data manualmente com `date -s` e sincronizando o relógio de hardware com `hwclock --systohc`.

Nesse incidente, o mesmo desync de relógio afetou a NOC-SERVER — só que dessa vez o efeito colateral foi mais sério: o desalinhamento de data/hora corrompeu as permissões do MariaDB (banco usado pelo Zabbix Server), impedindo consultas e gravações normais.

## Diagnóstico

1. Verificação do estado do Zabbix Server e Grafana — serviços "up", mas sem novos dados persistindo
2. Verificação do relógio da VM — confirmado desalinhamento significativo em relação ao horário real
3. Investigação do MariaDB — identificado problema de permissões associado ao salto de data/hora (timestamps de criação/expiração de sessão e credenciais afetados pela mudança abrupta de horário)

## Ação

1. Corrigido o relógio da VM manualmente (`date -s` + `hwclock --systohc`)
2. Corrigidas as permissões afetadas no MariaDB
3. Reiniciados os serviços do Zabbix Server e verificado o retorno da ingestão de dados no Grafana

## Resultado

Ingestão de dados normalizada; Zabbix e Grafana voltaram a exibir métricas em tempo real.

## Prevenção

- Configurar sincronização automática de horário (NTP/`systemd-timesyncd`) nas VMs, com verificação após qualquer pausa/reinício do host
- Adicionar um item de monitoramento no próprio Zabbix para alertar sobre desvio de relógio antes que ele afete o banco de dados (este ponto motivou diretamente o Incidente 3, documentado a seguir, que já usa esse tipo de alerta nativo do template Linux)
- Documentar como checklist de rotina: sempre validar `timedatectl` após retomar o ambiente de uma pausa prolongada
