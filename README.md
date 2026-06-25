# Case-PJ
# Análise de Satisfação de Clientes — CX PJ

**Relatório técnico do projeto** · Classificação de dores com IA + cruzamento de indicadores
Candidata: Nicolle Xavier dos Santos 

---

## 1. Objetivo

Transformar 4.529 comentários de clientes Pessoa Jurídica (PJ) — texto livre — em insumo de decisão,
combinando duas dimensões de análise:

1. **Quantitativa:** evolução da satisfação (nota média e % de detratores) por Comunidade, Produto e mês.
2. **Qualitativa:** o motivo central de cada comentário, resumido em **uma única palavra** ("dor") via IA.

A entrega cruza as duas dimensões para identificar **o que** move a satisfação, **por quê**, e **quais ações**
priorizar.

---

## 2. Dados de entrada

| Arquivo | Papel | Colunas principais |
|---|---|---|
| `dim_tarefa.xlsx` | Dimensão | `id_tarefa`, `tx_necessidade` (Produto), `Comunidade` |
| `fat_comentarios.xlsx` | Fato | `dt_resposta`, `cod_resp`, `id_tarefa`, `txt_coment`, `nota` (1–7) |

**Chave de relacionamento:** `id_tarefa`.
**Escopo:** 3 Comunidades (Contas, Pagamentos, Recebimentos), 5 Produtos, 5 meses (jan–mai/2026).

---

## 3. Arquitetura da solução

Pipeline único em Python (`analise_cx_pj.py`), em seis etapas sequenciais:

```
dados brutos
   │
   ├─ 1. Limpeza        → remove duplicidades (DISTINCT / drop_duplicates)
   ├─ 2. JOIN           → fato × dimensão por id_tarefa
   ├─ 3. Agregação      → nota média e % detratores por Comunidade/Produto/Mês
   ├─ 4. IA · Dores     → 1 palavra-dor por comentário (Claude Haiku)
   ├─ 5. Associação     → cruza evolução das dores × indicadores
   └─ 6. Exportação     → base classificada (.xlsx) + insumos (.json)
   │
recomendações priorizadas
```

---

## 4. Detalhamento técnico por etapa

### 4.1 Limpeza

Duas duplicidades tratadas antes de qualquer cálculo:

- **Dimensão:** `id_tarefa` 51867 aparecia duplicado → `drop_duplicates(subset="id_tarefa")` (5 Produtos únicos).
- **Fato:** 6 `cod_resp` repetidos → `drop_duplicates(subset="cod_resp")` (**4.535 → 4.529** respostas).

A data é normalizada para granularidade mensal (`Period("M")`) para permitir o agrupamento temporal.

```python
dim = dim.drop_duplicates(subset="id_tarefa")
fat = fat.drop_duplicates(subset="cod_resp")
fat["mes"] = pd.to_datetime(fat["dt_resposta"]).dt.to_period("M").astype(str)
```

### 4.2 JOIN (fato × dimensão)

`merge` à esquerda pela chave `id_tarefa`, anexando Produto e Comunidade a cada resposta. Equivalente
conceitual a um `PROCV`, aplicado em lote. Há uma asserção que garante 100% de correspondência (nenhum
`id_tarefa` órfão).

```python
df = fat.merge(dim, on="id_tarefa", how="left").rename(columns={"tx_necessidade": "Produto"})
assert df["Comunidade"].notna().all()
```

### 4.3 Agregação dos indicadores

Dois indicadores por grupo (Comunidade / Produto / mês):

- **Nota média** (1–7): termômetro geral.
- **% de detratores** (`nota <= 4`): mais sensível à insatisfação que a média.

A variação **jan→mai** (último mês − primeiro) responde objetivamente "quem melhorou/piorou".

```python
def add_indicadores(g):
    return pd.Series({
        "respostas":    len(g),
        "nota_media":   round(g["nota"].mean(), 2),
        "pct_detrator": round((g["nota"] <= 4).mean() * 100, 1),
    })
```

### 4.4 Classificação das dores com IA

**Abordagem em dois passos:**

1. **Descoberta da taxonomia** (1×, sobre amostra de 50 comentários): a IA propõe 8–10 temas recorrentes,
   sem inventar categorias ausentes. Resultado consolidado:
   `Instabilidade · Extrato · Lentidao · Limite · Comprovante · Pix · Usabilidade · Exportacao · Elogio`
   (mais `Outros` e `Sem_texto`).
2. **Classificação** (por comentário): prompt de sistema que força saída de **uma palavra**.

**Prompt de classificação (resumo):**

> Você é analista de CX de um banco (PJ). Leia o comentário e responda APENAS com UMA palavra que resuma a
> dor (ou elogio) central. Categorias reaproveitáveis; vago → `Outros`; vazio → `Sem_texto`.

**Chamada à API:**

```python
def classificar_dor_llm(texto):
    client = anthropic.Anthropic()
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",   # rápido e barato p/ 4,5k itens
        max_tokens=5,                          # força saída de 1 palavra
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": f'Comentário: "{texto}"\nDor:'}],
    )
    return msg.content[0].text.strip()
```

**Decisões de modelagem:**

| Decisão | Racional |
|---|---|
| Modelo Haiku | Menor custo/latência para alto volume de itens curtos. |
| `max_tokens = 5` | Restringe fisicamente a saída a uma palavra. |
| Cache por texto idêntico | Comentários repetidos (ex.: "App fora do ar") são classificados 1×. |
| Taxonomia emergente | Categorias derivadas dos dados → aderência à realidade PJ. |
| Batch em produção | Reduz custo em execução recorrente. |

**Reprodutibilidade — modo dual de execução:**

- **Com `ANTHROPIC_API_KEY`:** usa a IA real (Claude Haiku).
- **Sem chave:** usa um *fallback* de regras (palavras-chave) com a **mesma taxonomia**, permitindo reproduzir
  a análise de forma idêntica e sem custo. A lógica de negócio é invariante ao motor de classificação.

```python
USE_LLM = bool(os.environ.get("ANTHROPIC_API_KEY"))
classificador = classificar_dor_llm if USE_LLM else classificar_dor_regra
df["dor"] = df["txt_coment"].apply(classificar_cached)  # com cache
```

### 4.5 Associação (dor × indicador)

Para cada Comunidade, cruza-se a **distribuição e evolução mensal das dores** com a **trajetória dos
indicadores**, separando *causa* (dor estrutural crescente) de *incidente* (pico pontual).

### 4.6 Exportação

`base_classificada.xlsx` com as abas: `base_full`, `ind_comunidade_mes`, `ind_produto_mes`,
`dor_x_comunidade`, `dor_x_mes`. Adicionalmente, `insumos.json` para alimentar a apresentação. Toda saída é
auditável (comentário e dor lado a lado).

---

## 5. Resultados

### 5.1 Indicadores por Comunidade (jan → mai)

| Comunidade | Nota | Δ Nota | Detratores | Δ Detratores | Leitura |
|---|---|---|---|---|---|
| **Contas** | 5,68 → 4,67 | **−1,01** | 7,6% → 39,6% | **+32 pp** | Piora em todos os indicadores |
| **Pagamentos** | 5,72 → 5,69 | −0,03 | 8,2% → 9,9% | +1,7 pp | Estável; vale isolado em março |
| **Recebimentos** | 5,57 → 6,14 | **+0,57** | 4,3% → 0,0% | **−4,3 pp** | Melhora consistente |

### 5.2 Distribuição das dores nomeadas

| Dor | Qtd | | Dor | Qtd |
|---|---|---|---|---|
| Elogio | 957 | | Comprovante | 193 |
| Extrato | 304 | | Limite | 145 |
| Lentidão | 299 | | Usabilidade | 80 |
| Pix | 298 | | Exportação | 52 |
| Instabilidade | 250 | | | |

> `Outros` (1.870) e `Sem_texto` (81) representam ruído/vagueza e respostas vazias; mantidos fora das dores nomeadas.

### 5.3 Associação e diagnóstico

- **Contas — prioridade.** ~40% das dores nomeadas são *Extrato/Conciliação*, em crescimento mensal.
  Hipótese: novo formato de extrato consolidado quebrou a conciliação dos clientes PJ. A dor **explica** a
  queda do indicador.
- **Pagamentos — incidente, não tendência.** Vale isolado em março (detratores 41%, ~2× o normal) por
  *Instabilidade*, já revertido. Dor crônica de fundo: *Lentidão*.
- **Recebimentos — benchmark.** Melhora de ponta a ponta; boas práticas a replicar.

---

## 6. Recomendações priorizadas

| # | Comunidade | Ação |
|---|---|---|
| 1 | Contas | Reverter o formato do extrato (ou ofertar exportação OFX/CSV para conciliação) e validar com clientes-piloto. |
| 2 | Pagamentos | Monitorar estabilidade e atacar a lentidão estrutural (performance do app). |
| 3 | Recebimentos | Mapear e replicar as boas práticas nas demais Comunidades. |
| 4 | Transversal (IA) | **Painel de escuta contínua:** classificar comentários em tempo real (mesmo prompt de 1 palavra) e alertar quando uma dor ultrapassar limiar mensal. Antecipa crises. |

---

## 7. Como executar

### Pré-requisitos

```bash
pip install pandas openpyxl anthropic
```

### Configuração de caminhos

No topo de `analise_cx_pj.py`, ajustar para o local dos arquivos. Se estiverem na mesma pasta do script:

```python
UP  = "."   # pasta com dim_tarefa.xlsx e fat_comentarios.xlsx
OUT = "."   # destino das saídas
```

### Execução

```bash
# Sem IA (fallback de regras — reproduzível, sem custo)
python analise_cx_pj.py

# Com IA real (Claude Haiku)
export ANTHROPIC_API_KEY="sua_chave"   # Windows: set ANTHROPIC_API_KEY=...
python analise_cx_pj.py
```

### Saídas geradas

- `base_classificada.xlsx` — base completa + tabelas agregadas (5 abas).
- `insumos.json` — agregados para a apresentação.

---

## 8. Decisões de engenharia (resumo)

| Tema | Escolha | Benefício |
|---|---|---|
| Organização | Script único, comentado | Reprodutibilidade ponta a ponta |
| Integridade | `drop_duplicates` + `assert` no JOIN | Números confiáveis |
| Indicador | Nota média **e** % detratores | Captura insatisfação que a média esconde |
| IA | Prompt de 1 palavra + `max_tokens=5` | Saída objetiva e classificável |
| Custo | Cache + batch + Haiku | Viável em escala recorrente |
| Robustez | Modo dual (IA / regras) | Roda com ou sem API, sem alterar a análise |
