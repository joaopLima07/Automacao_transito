# Workflow: Transito — Cobranca de Notas Fiscais em Transito

Workflow n8n responsavel por identificar Notas Fiscais (NF) de transferencia entre filiais que estao em transito ha mais de 20 dias sem entrada conferida, e notificar automaticamente as lojas responsaveis por e-mail.

---

## Visao Geral

O workflow possui **duas secoes principais**, ambas disparadas pelo mesmo gatilho semanal (toda segunda-feira as 07h00):

| Secao | Descricao |
|---|---|
| **Resumo da ultima semana do mes** | Envia um resumo consolidado de todas as NF em transito, somente se for a ultima segunda-feira do mes |
| **Cobrancas normais** | Envia e-mails de cobranca para cada filial com NF pendente, toda semana |

---

## Gatilho

- **Tipo:** Schedule Trigger (Cron)
- **Nome:** `Executa apenas nas segundas`
- **Expressao Cron:** `0 0 07 * * 1` toda segunda-feira as 07:00
- **Verificacao adicional:** O no `Ultima segunda do mes` avalia se a data atual e a ultima segunda do mes para acionar o fluxo de resumo

---

## Banco de Dados (Microsoft SQL Server)

### Tabelas utilizadas

| Tabela | Descricao |
|---|---|
| `LOJA_ENTRADAS` | Registros de entradas de NF nas filiais |
| `TRANSITO_ATUALIZAVEL` | Controle de envio de cobrancas por NF |
| `TB_FILIAIS_TRANSITO` | Cadastro de filiais ativas com e-mails de contato |

### Logica das queries

A query principal busca NFs da tabela `LOJA_ENTRADAS` com join em `TRANSITO_ATUALIZAVEL`, filtrando registros com mais de 20 dias desde a emissao, entrada nao conferida (`ENTRADA_CONFERIDA = 0`) e sem data de envio registrada (`data_envio IS NULL`), ordenando pela emissao mais antiga primeiro.

A query de filiais busca todas as filiais ativas (`inativo = 0`) da tabela `TB_FILIAIS_TRANSITO`, retornando o codigo da filial e os e-mails do gerente e supervisor.

---

## Fluxo de Execucao

### Secao 1 — Resumo da ultima semana do mes