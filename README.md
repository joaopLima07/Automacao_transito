# Workflow: Transito — Cobranca de Notas Fiscais em Transito

Workflow n8n responsavel por identificar Notas Fiscais (NF) de transferencia entre filiais que estao em transito ha mais de 20 dias sem entrada conferida, e notificar automaticamente as lojas responsaveis por e-mail.

---

## Visao Geral

O workflow possui **duas secoes principais**, ambas disparadas pelo mesmo gatilho semanal (toda segunda-feira as 07h00):

| Secao | Descricao |
|---|---|
| **Resumo da ultima semana do mes** | Envia um resumo consolidado de todas as NF em transito, mas **somente se for a ultima segunda-feira do mes** |
| **Cobrancas normais** | Envia e-mails de cobranca para cada filial com NF pendente, toda semana |

---

## Gatilho

- **Tipo:** Schedule Trigger (Cron)
- **Nome:** `Executa apenas nas segundas`
- **Expressao Cron:** `0 0 07 * * 1` toda **segunda-feira as 07:00**
- **Verificacao adicional:** O no `Ultima segunda do mes` avalia se a data atual e a ultima segunda do mes para acionar o fluxo de resumo

---

## Banco de Dados (Microsoft SQL Server)

### Tabelas utilizadas

| Tabela | Descricao |
|---|---|
| `LOJA_ENTRADAS` | Registros de entradas de NF nas filiais |
| `TRANSITO_ATUALIZAVEL` | Controle de envio de cobrancas por NF |
| `TB_FILIAIS_TRANSITO` | Cadastro de filiais ativas com e-mails de contato |

### Query principal (NFs em atraso)

\`\`\`sql
SELECT a.filial
     , a.filial_origem
     , a.numero_nf_transferencia
     , a.emissao
     , a.valor_total
FROM LOJA_ENTRADAS a
    LEFT JOIN TRANSITO_ATUALIZAVEL b
        ON  a.filial                  = b.filial_destino
        AND a.FILIAL_ORIGEM           = b.filial_origem
        AND a.NUMERO_NF_TRANSFERENCIA = b.numero_NF
WHERE a.emissao < DATEADD(day, -20, GETDATE())
  AND a.ENTRADA_CONFERIDA = 0
  AND a.filial = '{{ $json.filial }}'
  AND b.data_envio IS NULL
ORDER BY a.emissao ASC
\`\`\`

### Query de filiais ativas

\`\`\`sql
SELECT filial, email_loja, email_supervisor
FROM TB_FILIAIS_TRANSITO
WHERE inativo = 0
\`\`\`

---

## Fluxo de Execucao

### Secao 1 — Resumo da ultima semana do mes

\`\`\`
Trigger (segunda 07h)
  → Verifica se e a ultima segunda do mes
    → [SIM] Busca todas as filiais ativas (TB_FILIAIS_TRANSITO)
        → Loop por filial
            → Retorna e-mail de destino (gerente ou supervisor)
              → Query NFs em atraso por filial (LOJA_ENTRADAS x TRANSITO_ATUALIZAVEL)
                → Verifica se ha NFs pendentes (IF)
                  → [SIM] Agrega dados → Filtra valor != 0 → Envia e-mail de cobranca
                  → [NAO] Envia e-mail notificando que todas as notas foram cobradas
\`\`\`

### Secao 2 — Cobrancas normais (toda semana)

\`\`\`
Trigger (segunda 07h)
  → Busca todas as filiais ativas (TB_FILIAIS_TRANSITO)
      → Loop por filial
          → Retorna e-mail de destino (gerente ou supervisor)
            → Query NFs em atraso (sem filtro de data_envio)
              → Verifica se ha NFs pendentes (IF)
                → [SIM] Agrega dados → Filtra valor != 0 → Envia e-mail de cobranca
                              → Chama sub-workflow: TRANSITO_ATUALIZAVEL (registra envio)
                → [NAO] Envia e-mail notificando que todas as notas foram cobradas
\`\`\`

---

## Notificacoes por E-mail (Microsoft Outlook)

### Logica de destinatario

O e-mail e enviado ao **gerente da loja** (`email_loja`). Caso a loja possua um supervisor cadastrado (`email_supervisor`), ele e adicionado em copia.

### Tipos de e-mail enviados

| Tipo | Quando | Destinatario |
|---|---|---|
| E-mail de cobranca | Quando ha NFs em transito sem entrada | Gerente + Supervisor da filial |
| E-mail de conclusao | Quando todas as NFs foram cobradas | Equipe interna |

### Filiais monitoradas

- **IB** (Inbrands)
- **TH** (outra filial)
- Demais filiais ativas na tabela `TB_FILIAIS_TRANSITO`

---

## Sub-workflow

| Sub-workflow | ID | Funcao |
|---|---|---|
| `Atualiza tabela TRANSITO_ATUALIZAVEL` | `Tm4r5G7GrUkK9cqz` | Registra a data de envio da cobranca para evitar reenvios |

---

## Nos utilizados (n8n)

| Tipo de no | Quantidade | Funcao |
|---|---|---|
| `Microsoft SQL` | 8 | Consultas ao banco de dados |
| `Microsoft Outlook` | 8 | Envio de e-mails |
| `Loop Over Items` (Split in Batches) | 4 | Iteracao por filial |
| `If` | 8 | Condicionais de fluxo |
| `Aggregate` | 4 | Consolidacao de dados por filial |
| `Edit Fields` (Set) | 10+ | Mapeamento e formatacao de campos |
| `Code` (JavaScript) | 6 | Filtros customizados e logica de e-mail |
| `Execute Workflow` | 2 | Chamada ao sub-workflow |
| `Schedule Trigger` | 1 | Gatilho semanal |
| `Sticky Notes` | 14 | Documentacao visual no canvas |

---

## Estrutura dos Dados Processados

\`\`\`json
{
  "filial": "IB",
  "filial_origem": "SP",
  "numero_nf_transferencia": "000123",
  "emissao": "2024-10-01",
  "valor_total": 1500.00
}
\`\`\`

---

## Pre-requisitos

- **n8n** instalado e configurado
- Credenciais configuradas:
  - `Microsoft SQL Server` — acesso as tabelas `LOJA_ENTRADAS`, `TRANSITO_ATUALIZAVEL`, `TB_FILIAIS_TRANSITO`
  - `Microsoft Outlook` — conta com permissao de envio
- Sub-workflow `Atualiza tabela TRANSITO_ATUALIZAVEL` (ID: `Tm4r5G7GrUkK9cqz`) publicado e ativo

---

## Como importar

1. Acesse o n8n → **Workflows** → **Import from file**
2. Selecione o arquivo `Transito.json`
3. Configure as credenciais de SQL Server e Outlook
4. Ative o workflow

---

## Observacoes

- NFs sao consideradas em atraso quando tem **mais de 20 dias** desde a emissao e `ENTRADA_CONFERIDA = 0`
- NFs com `valor_total = 0` sao **filtradas** e nao geram cobranca
- O campo `data_envio` na tabela `TRANSITO_ATUALIZAVEL` evita cobrancas duplicadas no fluxo normal
- O resumo da ultima semana do mes **nao filtra** por `data_envio`, exibindo todas as pendencias independentemente de cobrancas anteriores