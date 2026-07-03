# Tratamento da Base de Vendas Porsche

Agente determinístico de limpeza para a base bruta de vendas de veículos Porsche
(`db_psc_25354543_no_tracked`). O agente segue, coluna a coluna, o contrato de
limpeza definido em [`docs/schema.md`](docs/schema.md): normaliza formatos,
desambigua valores e **quarentena** (nunca "chuta") qualquer campo irrecuperável,
mantendo a linha na base limpa.

> **Grão:** 1 linha = 1 venda de veículo · **100 linhas** · **12 colunas brutas**
> → **15 colunas tratadas**. Domínio: carros novos/seminovos, preços ~US\$ 58,9k–286,5k,
> anos-modelo 2020–2026.

---

## Estrutura do repositório

```
tratamento-vendas-porsche/
├── README.md                                  # este arquivo
├── dashboard.html                             # dashboard de verificação (abre no navegador)
├── requirements.txt                           # dependências (pandas, openpyxl)
├── .gitignore
├── data/
│   ├── raw/
│   │   └── db_psc_25354543_no_tracked.xlsx    # entrada bruta (não tratada)
│   └── processed/
│       └── base_porsche_limpa.xlsx            # saída final tratada (4 abas)
├── docs/
│   └── schema.md                              # contrato de limpeza (regras por coluna)
└── src/
    └── agente_limpeza_porsche.py              # o agente
```

---

## Como executar

```bash
# 1. instalar dependências
pip install -r requirements.txt

# 2. rodar o agente: entrada -> saída
python src/agente_limpeza_porsche.py \
       data/raw/db_psc_25354543_no_tracked.xlsx \
       data/processed/base_porsche_limpa.xlsx
```

O script imprime o relatório de limpeza no terminal e grava a planilha de saída
com quatro abas:

| Aba          | Conteúdo                                                        |
|--------------|-----------------------------------------------------------------|
| `base_limpa` | base final tratada (schema §4); campos quarentenados = `null`   |
| `auditoria`  | valor bruto (`_raw`) vs. valor limpo, lado a lado (schema §2.5)  |
| `quarentena` | 1 linha por campo irrecuperável: `sale_id, column, raw_value, reason` |
| `relatorio`  | validações do schema §5 (cardinalidades, faixas, contagens)     |

---

## Dashboard de verificação

O arquivo [`dashboard.html`](dashboard.html) é um **console de qualidade de dados**
self-contained: os dados tratados já vêm embutidos, então **é só abrir no
navegador** (duplo-clique) — não precisa de servidor, build nem internet.

O que ele mostra:

- **Velocímetro de integridade** — % de linhas 100% limpas (recalcula conforme os filtros).
- **KPIs** — receita total, ticket médio, faixa de preço, milhagens convertidas de KM, milhagens suspeitas.
- **Gráficos** — vendas por linha de modelo, receita por mês, forma de pagamento, status de entrega, distribuição de preço e top estados.
- **Tabela de registros** — 15 colunas, ordenável, com busca por cliente/vendedor/cidade e filtros por linha, pagamento, entrega e UF. Campos quarentenados aparecem como `null` em âmbar; milhagens suspeitas ganham um marcador.
- **Log de quarentena** — os 21 campos zerados, com valor bruto e motivo.

Para regenerar o dashboard depois de rodar o agente numa base nova, reabra o
`dashboard.html` — os dados são lidos do próprio HTML, então basta gerar um novo
a partir da `base_limpa` (ver comentário no topo do arquivo).

### Publicar no GitHub Pages (opcional)

Em **Settings → Pages**, selecione a branch `main` e a pasta raiz (`/root`).
O dashboard fica acessível em
`https://<seu-usuario>.github.io/tratamento-vendas-porsche/dashboard.html`.

---

## O que o agente faz

O agente é **determinístico**: não usa heurística "esperta" nem adivinha valores.
Cada coluna tem um parser próprio, com ordem de parsing obrigatória, e um
_sanity check_ final. Se o valor não é recuperável com segurança, o **campo** vira
`null` e a linha vai para a tabela de quarentena com o motivo — a **linha inteira
nunca é descartada**.

### Regras por coluna (resumo)

- **`sale_date`** *(mais problemática)* — usa o serial do Excel direto quando já é
  data; senão parseia por família de formato (ISO, pontuado, US `MM/DD/YYYY`,
  extenso, ordinal). Aplica a regra **"mês > 12 ⇒ a ordem é dia/mês"**
  (`2024/15/07` → `2024-07-15`), interpreta ano de 2 dígitos como `20YY`, e
  **valida o calendário real** (dia do mês + ano bissexto). Data logicamente
  impossível (`2024-02-30`, `February 29th, 2025`) ⇒ `null` + quarentena
  `data_invalida`. Fora de 2024–2027 ⇒ `data_fora_de_faixa`.
- **`sale_price`** — remove rótulos (`$`, `USD`, `dollars`); trata sufixo `k`
  (×1000); desambigua **formato US vs. europeu** pelo separador mais à direita
  (`$89.750,00` → `89750.00`, `$79,500.00` → `79500.00`); entende preço por
  extenso (`eighty two thousand`). Sanity check final em **US\$ 30k–400k**.
- **`vehicle_mileage`** — normaliza `zero`/`new` → 0, extenso (`twelve thousand`
  → 12000), milhar europeu; **converte KM → milhas** (`km / 1.60934`) preservando
  a origem em `mileage_unit_raw`; sinaliza valores brutos `< 100` sem contexto com
  a flag `mileage_suspect`.
- **`model_year`** — inteiro, extenso (`twenty twenty four`), com espaço/hífen
  (`20 23`, `20-25`) → `int`. Faixa 2020–2026.
- **`payment_method`** / **`delivery_status`** — colapsa dezenas de variações
  (caixa, hífen, underscore, pontuação, typos como `DELIVERD`) nas categorias
  canônicas do schema.
- **`state`** — nome completo ou sigla → sigla **USPS** de 2 letras, validada
  contra a lista oficial dos 50 estados.
- **`customer_name` / `city` / `salesperson`** — Title Case preservando hifens
  (`Daniel-Jones`) e abreviações (`St. Louis`). Nomes não são "completados".
- **`porsche_model`** — normalização de espaços/caixa + derivação de `model_line`
  (718 / 911 / Taycan / Panamera / Macan / Cayenne).

---

## Resultado da execução

Rodando na base bruta (`data/raw/`):

| Métrica                                    | Valor           |
|--------------------------------------------|-----------------|
| Linhas totais                              | 100             |
| Linhas 100% limpas                         | **79**          |
| Linhas com ≥1 campo quarentenado           | 21              |
| `sale_id` únicos / nulos                   | 100 / 0         |
| Datas parseadas OK                         | 79 (79,0%)      |
| Datas em quarentena (`data_invalida`)      | 21              |
| Preço — faixa observada                    | US\$ 58.900 – 286.500 |
| Preços fora de faixa (30k–400k)            | 0               |
| `model_year` fora de faixa                 | 0               |
| Estados inválidos                          | 0               |
| Milhagens convertidas de KM                | 4               |
| `mileage_suspect = true`                   | 4 (IDs 9, 16, 40, 90) |
| `payment_method` "other" inesperado        | 0               |
| `delivery_status` "unknown" inesperado     | 0               |

**As 21 quarentenas são todas de `sale_date`** — exatamente as datas
logicamente impossíveis da base (dia 30/02, dia 40, 29/02 em ano não bissexto,
Nov 31, etc.). Nenhuma data válida foi quarentenada por engano, e nenhum outro
campo precisou de quarentena.

### Cardinalidade final das categorias

- **`payment_method`**: `bank_transfer` (40), `credit_card` (15), `financing` (13),
  `cash` (12), `lease` (10), `crypto` (6), `debit_card` (4).
- **`delivery_status`**: `delivered` (41), `pending` (24), `in_transit` (19),
  `awaiting` (9), `cancelled` (7).

---

## Decisões de agrupamento (do schema, a confirmar)

Duas escolhas seguem o _default_ do schema e podem ser ajustadas no dicionário
correspondente em `src/agente_limpeza_porsche.py`:

1. `wire` / `bank wire` / `ACH payment` foram agrupados em **`bank_transfer`**.
   Para separar ACH de wire, quebrar o dicionário `_PAYMENT`.
2. `awaiting delivery` / `awaiting pickup` ficaram como categoria **`awaiting`**,
   separada de `pending`. Para unir, remapear no dicionário `_DELIVERY`.

---

## Princípio de quarentena

> Quarentena zera **só o campo**, não a linha. A venda continua na base limpa com
> os demais campos tratados. Nada de adivinhar — data `30/02` vira `null` +
> registro na quarentena, ponto.

Exemplo: a venda `sale_id = 6` tinha `sale_date = 2024-02-30` (impossível). Na
`base_limpa` a data fica `null` (destacada em amarelo), mas preço, modelo, estado
e forma de pagamento permanecem tratados e a linha segue na base.
