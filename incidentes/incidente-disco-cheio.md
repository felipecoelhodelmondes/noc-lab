# Incidente 2 — Disco cheio na NOC-CLIENT

**Tipo:** Simulado (controlado)
**Host afetado:** NOC-CLIENT
**Categoria:** Performance / Disponibilidade

## Sintoma

Uso de disco (`/`) subindo rapidamente até aproximadamente 97% de ocupação, disparando alerta de espaço em disco no Zabbix.

## Impacto

Disco quase saturado compromete gravação de logs, cache de aplicações e pode levar o sistema a ficar inoperante (falha ao escrever arquivos temporários, travamento de serviços). Em produção, esse cenário costuma preceder indisponibilidade total do host.

## Reprodução (simulação controlada)

O cenário foi gerado propositalmente com `fallocate`, criando um arquivo grande o suficiente para forçar o uso do disco a ~97%:

```bash
fallocate -l <tamanho> /caminho/arquivo_teste
```

## Diagnóstico

1. Zabbix disparou trigger de espaço em disco baixo no host NOC-CLIENT
2. Painel **Disk** do dashboard Grafana (NOC-CLIENT) capturou visualmente o salto de uso de disco em tempo real
3. Confirmado no host, via `df -h`, que a partição `/` estava em ~97% de uso

## Ação (primeiro combate N1)

1. Identificado o arquivo criado artificialmente pelo teste
2. Arquivo removido para liberar espaço:
   ```bash
   rm /caminho/arquivo_teste
   ```
3. Confirmado o retorno do uso de disco a níveis normais via `df -h` e no painel Disk do Grafana

## Resultado

Alerta de disco cheio resolvido no Zabbix automaticamente após a liberação de espaço; painel Disk do Grafana voltou a mostrar uso normal.

## Prevenção

- Configurar múltiplos thresholds de alerta (ex: 80% aviso, 90% crítico) para dar tempo de reação antes da saturação total
- Rotina periódica de limpeza de logs antigos e arquivos temporários
- Monitorar tendência de crescimento de disco (não só o valor absoluto) para prever saturação antes que ela aconteça
