# 🧬 Chunking Semântico com Abordagem Híbrida para Busca em Catálogo Farmacêutico com LLM

> Trabalho da disciplina de **Extração e Preparação de Dados**  
> Empresa: ACHÉ Laboratórios Farmacêuticos  
> Modelo utilizado na validação: **Gemini** (via Gem)

---

## 📋 Sobre o Projeto

Este projeto implementa uma estratégia de **chunking semântico** aplicada ao catálogo de produtos da ACHÉ Laboratórios Farmacêuticos, com o objetivo de viabilizar a identificação automática de medicamentos por uma **LLM de pequeno porte (até 4B parâmetros)**.

O sistema processa e-mails com solicitações de orçamento em linguagem natural e gera automaticamente um **JSON estruturado** com os produtos identificados, preços calculados por ICMS/UF e valor total.

---

## 🎯 Problema

A ACHÉ recebe diariamente e-mails de clientes solicitando orçamentos. Esses e-mails são informais e sem padronização — os clientes referenciam produtos de três formas distintas:

| Tipo de Busca | Exemplo |
|---|---|
| Por **nome comercial** | `10x ACICLOVIR 200MG COM BLX25` |
| Por **código SAP** | `20x 1003649` |
| Por **princípio ativo** | `RACECADOTRILA x20` |

O sistema deve ainda identificar o **tipo de cliente** (PF, PJ ou Revenda) e a **UF**, para aplicar a alíquota de ICMS correta e gerar o preço final.

---

## 💡 Solução: Abordagem Híbrida

A estratégia adotada divide cada produto do catálogo em **dois objetos complementares**:

```
1 linha do CSV  →  1 par de chunks (Objeto 1 + Objeto 2)
```

### Objeto 1 — Texto Narrativo (para busca semântica)
Gerado por **template Python determinístico** — sem LLM, 100% de acurácia.  
Destinado à vetorização por embedding e busca semântica.

```
O produto ACICLOVIR 200MG COM BLX25, identificado pelo código SAP 1006459,
é um medicamento genérico fabricado pela ACHÉ. Seu princípio ativo é o aciclovir,
apresentado na forma de comprimido com 25 unidades por embalagem. Pertence ao
segmento de genéricos, com tarja vermelha, lista positiva RENAME e regime monitorado.
```

### Objeto 2 — JSON Estruturado (para cálculo de preços)
Contém todos os metadados do produto e a **tabela completa de preços por ICMS**.  
Recuperado por lookup exato via `codigo_sap` — nunca vetorizado.

```
CHUNK ID: CHK-1006459

[1] IDENTIFICAÇÃO
  codigo_sap   : 1006459
  familia      : ACICLOVIR
  apresentacao : ACICLOVIR 200MG COM BLX25

[2] COMPOSIÇÃO
  principio_ativo    : ACICLOVIR
  forma_farmaceutica : comprimido
  quantidade         : 25 COM

[3] CLASSIFICAÇÃO
  tipo_medicamento : GENÉRICO
  tarja            : TV (tarja vermelha)
  regime_precos    : MONITORADO
  empresa          : ACHÉ

[4] TRIBUTAÇÃO
  ncm: 30049069  |  ipi: 0%  |  pis: 0%  |  cofins: 0%

[5] PREÇOS POR UF E TIPO DE CLIENTE
  ICMS  12% | Pessoa Física  | SP, MG, PR, SC, RS       | R$ 93,67
  ICMS  12% | PMC/Farmácia   | SP, MG, PR, SC, RS       | R$ 129,49
  ICMS  17% | Pessoa Física  | RJ, GO, MT, MS e outros  | R$ 94,24
  ICMS  17% | PMC/Farmácia   | RJ, GO, MT, MS e outros  | R$ 130,28
  ICMS  20% | PMC/Farmácia   | BA, DF, RJ e outros      | R$ 134,36

[6] TEXTO NARRATIVO
  O produto ACICLOVIR 200MG COM BLX25 ...
```

---

## 🔄 Fluxo do Sistema RAG

```
E-mail do cliente
      │
      ▼
[LLM] Extrai: produtos, quantidades, UF, tipo de cliente
      │
      ├── Termo NUMÉRICO (código SAP)?
      │       └── Lookup direto no Objeto 2 → 100% preciso
      │
      └── Termo TEXTUAL (nome / princípio ativo)?
              └── Embedding → busca semântica no Objeto 1
                      └── Extrai codigo_sap → Lookup no Objeto 2
      │
      ▼
[Sistema] Seleciona ICMS correto: UF + tipo de cliente
      │
      ▼
[LLM] Monta e retorna JSON de orçamento
```

---

### Expansão de abreviações

| Abreviação | Forma Expandida |
|---|---|
| `COM` | comprimido |
| `COM REV` | comprimido revestido |
| `CAP` | cápsula |
| `FR` | frasco |
| `BG` | bisnaga |
| `XPE` | xarope |
| `TV` | tarja vermelha |
| `VL` | sem tarja (venda livre) |
| `TP` | tarja preta |

---

## 🧪 Validação no Gemini

### Como testar

1. Abra o [Gemini](https://gemini.google.com) e crie um **Gem**
2. Cole o conteúdo de `System_Prompt.txt` como instrução do Gem
3. Anexe o arquivo `chunks_narrativos_ACHEv4.txt` como base de conhecimento
4. Execute o "Casos de teste" para testar

### Casos de teste

#### Teste 1 — Busca por nome
```
Olá, gostaria de receber um orçamento:
10x ACICLOVIR 200MG COM BLX25
10x ALENIA 12/400MCG POINAL CAP FRX60+INAL
JoseFarmacia — Rio de Janeiro — RJ
```

**JSON esperado:**
```json
{
  "itens": [
    {
      "produto": "ACICLOVIR 200MG COM BLX25",
      "codigo_sap": "1006459",
      "quantidade": 10,
      "preco_unitario": 130.28,
      "preco_total": 1302.80,
      "icms_aplicado": "20%",
      "tipo_preco": "PMC"
    },
    {
      "produto": "ALENIA 12/400MCG POINAL CAP FRX60+INAL",
      "codigo_sap": "1006368",
      "quantidade": 10,
      "preco_unitario": 158.80,
      "preco_total": 1588.00,
      "icms_aplicado": "20%",
      "tipo_preco": "PMC"
    }
  ],
  "cliente": { "tipo": "Pessoa Jurídica", "uf": "RJ", "icms_regra": "ICMS 20% PMC" },
  "total": 2890.80
}
```

#### Teste 2 — Busca por código SAP
```
Me enviem o orçamento:
10x 1000001 | 20x 1003649 | 30x 1006490 | 40x 1003648
Pessoa física — São Paulo — SP
```

#### Teste 3 — Busca por princípio ativo
```
Me enviem o orçamento:
CLORIDRATO DE ISOTIPENDIL x10
RACECADOTRILA x20
SULFATO DE BLEOMICINA x30
Revenda — Salvador — BA
```

## 📐 Por que Abordagem Híbrida?

| Critério | JSON Puro | Texto Narrativo Puro | **Híbrido** |
|---|---|---|---|
| Busca semântica | ⚠️ Média | ✅ Alta | ✅ Alta |
| Busca por código exato | ✅ Alta | ⚠️ Média | ✅ Alta |
| Precisão no cálculo de preço | ✅ Alta | ❌ Frágil | ✅ Alta |
| Performance com LLM pequena | ⚠️ Média | ✅ Alta | ✅ Alta |

---


> **Disciplina:** Extração e Preparação de Dados · **Ano:** 2025
