# Tech Challenge Fase 3: Big Data to Analytics

**PosTech FIAP Data Analytics**
Pipeline AWS sobre a pesquisa **State of Data Brasil** (Data Hackers + Bain),
três edições, arquitetura em camadas Bronze / Silver / Gold.

---

## O cenário

Uma consultoria estratégica em dados foi contratada por uma **instituição
financeira de grande porte** que quer expandir sua área de Dados, Analytics e
IA. Antes de definir estratégia de contratação, capacitação e investimento, ela
precisa entender o cenário atual do mercado brasileiro.

A fonte são as três últimas edições do State of Data Brasil. Os dados chegam
brutos: 399, 403 e 388 colunas, esquemas que mudam de um ano para o outro, e
nenhuma coluna que diga a qual edição cada resposta pertence.

---

## Arquitetura

```
Kaggle (3 edições)
   │  ingestão via AWS CLI / Boto3
   ▼
S3 · BRONZE          3 tabelas separadas, sem join, Parquet
   │                 cópia FIEL da origem (SOR): nome de coluna, acento e
   │                 espaço como vieram do Kaggle. Tudo string.
   │  Glue Job (PySpark)
   ▼
S3 · SILVER          união harmonizada das 3 edições (SOT)
   │                 normaliza, aplica o de-para e tipa
   │                 14.002 linhas × 79 colunas, 1 por respondente por edição
   │  Glue Job (PySpark)
   ▼
S3 · GOLD            modelo dimensional (SPEC)
   │                 1 fato + 11 dimensões + catálogo + bridge + 2 desconectadas
   ▼
Athena  →  consultas analíticas  →  results/*.csv
   │
   │  exportação da Gold para CSV
   ▼
Power BI  →  tc3_state_of_data.pbix, 15 telas

Glue Data Catalog cobre as TRÊS camadas (é transversal, não etapa final)
```

Diagrama completo em [`docs/`](docs/).

---

## Estrutura do repositório

| Pasta | O que tem |
|---|---|
| `apresentacao/` | **entregável 1**: o material executivo em .pptx e .pdf, 25 slides |
| `notebooks/` | **entregável 3**: os cinco notebooks, ingestão a gráficos |
| `glue/` | o código que roda na AWS, exatamente como está lá |
| `sql/` | DDL de catalogação e as consultas do Athena por pergunta |
| `powerbi/` | o `.pbix` do dashboard executivo e o PDF das 15 telas |
| `docs/` | **entregável 2**: o diagrama da arquitetura, mais as respostas das 7 perguntas, modelo, medidas, validações e decisões |
| `results/` | CSV de cada consulta do Athena e os 16 gráficos em `graficos/` |
| `data/amostras/` | amostra pequena para inspeção (a base vem do Kaggle) |
| `dev/` | testes e utilitários de desenvolvimento, não fazem parte do pipeline |

### Por que `glue/` tem mais de um arquivo por camada

O Glue aceita um script principal mais módulos auxiliares via `--extra-py-files`.
As regras de negócio ficam separadas do código que as executa, o que permite
trocar um de-para sem tocar no job:

```
glue/silver/
  job_silver.py                 o script do Glue Job
  config_silver.py              de-para de colunas, categorias e tipos
  grupos_multipla_escolha.py    quais colunas formam cada pergunta

glue/gold/
  job_gold.py                   o script do Glue Job
  gold_comum.py                 leitura da Silver e utilitários de denominador
  dim_rotulos.py                rótulos de exibição e ordenação
  analises/                     as 85 tabelas agregadas (conferência)
```

---

## As três camadas

### Bronze (Gusthavo)

Cópia fiel da origem. Três tabelas separadas, sem join, sem tipagem, sem
enriquecimento. Parquet particionado por ano e catalogado no Glue.

`inferSchema=false` de propósito: a inferência converte colunas binárias de
texto (`"0"`/`"1"`) para float (`0.0`/`1.0`), o que quebra o join entre edições
e viola a regra da camada. Afetava 328 de 399 colunas.

### Silver (Caio)

União harmonizada. **14.002 linhas × 79 colunas**, uma por respondente por
edição, particionada por `ano_pesquisa`.

Resolve o que a Bronze não pode:

1. **injeta `ano_pesquisa`** a partir da origem do arquivo, porque nenhuma
   edição traz coluna de período. Validado contra a data de envio real;
2. **cria `sk_respondente`**, já que a chave primária muda de nome entre
   edições (`id` em 2023-2024, `token_user` nas outras);
3. **normaliza booleano**: seis colunas vêm `TRUE`/`FALSE` só em 2024-2025 e
   `0`/`1` nas outras duas;
4. **consolida ~320 colunas binárias** em 17 grupos de múltipla escolha;
5. **marca `serie_comparavel`** nas 1.282 linhas com categoria que não existe
   nas três edições.

Contrato completo em [`docs/CONTRATO_SILVER.md`](docs/CONTRATO_SILVER.md).

### Gold (Caio)

Um star schema só, que responde as sete perguntas. **16 tabelas**: um fato no
grão de respondente, 11 dimensões conformadas, um catálogo de opções, uma bridge
que liga o fato às múltiplas escolhas e duas tabelas desconectadas.

Uma das dimensões, `dim_benchmark`, traz **referências externas de mercado**
(Bain, Cetic.br, Brasscom, CAGED, Peers/MIT) para contextualizar os números da
pesquisa. Cada uma conferida na fonte primária, com população, amostra e
ressalva declaradas. Ver [`docs/BENCHMARK_EXTERNO.md`](docs/BENCHMARK_EXTERNO.md).

Modelo em [`docs/MODELO_GOLD.md`](docs/MODELO_GOLD.md).

---

## ⚠️ Três armadilhas desta base

Todas produzem número errado **sem gerar erro**. Estão tratadas no código e
documentadas, mas quem for escrever consulta nova precisa conhecer.

**1. `uf` muda de significado entre edições.** Em 2023-2024 e 2024-2025 é onde
a pessoa mora; em 2025-2026 é onde ela nasceu (100% de match com
`estado_origem`). A Silver ignora essa coluna e deriva a sigla de `estado`.

**2. O denominador quase nunca é 14.002.** Vários blocos são condicionais:
`ia_gen_prioridade` só é respondido por gestores (2.593), e
`empresa_possui_datalake` só por engenheiros de dados (2.396). A interseção
entre os dois é **zero**. Calcular percentual sobre a base inteira devolve 3%
onde o número real é 61%.

**3. `NULL` não é zero na múltipla escolha.** Quem não viu a pergunta fica
`NULL`; quem viu e não marcou fica `''` e `0`. Filtre `IS NOT NULL` antes de
dividir.

---

## As sete perguntas

| # | Pergunta | Resposta |
|---|---|---|
| 1 | Como está estruturado o mercado brasileiro de Dados? | [`RESPOSTAS_P1_A_P6.md`](docs/RESPOSTAS_P1_A_P6.md) |
| 2 | Quais perfis profissionais são mais valorizados? | idem |
| 3 | Qual é o cenário de diversidade de gênero? | idem |
| 4 | Quais tecnologias têm maior adoção? | idem |
| 5 | Qual é o índice de adoção de IA e seu impacto? | idem |
| 6 | Há diferenças entre regiões, senioridades ou modelos de trabalho? | idem |
| 7 | Quais oportunidades e desafios para quem quer investir em Dados e IA? | [`RESPOSTA_P7.md`](docs/RESPOSTA_P7.md) |

Alguns achados:

- a adoção individual de IA chegou a **97,9%** em 2025, mas só **60,6%** das
  empresas a tratam como prioridade. Um gap de 38 pontos;
- **71%** das barreiras à IA apontadas por gestores são de capital humano
  (falta de expertise e de compreensão do caso de uso), não de tecnologia;
- a participação feminina **caiu** de 24,4% para 22,0% em dois anos, e o gap
  salarial no nível sênior **subiu** de 10,9% para 19,0%;
- **94%** de quem nasce no Norte trabalha fora da região;
- o modelo de trabalho pesa mais que a região: sênior remoto ganha **18,6%** a
  mais que presencial, controlado por senioridade.

E duas validações contra fonte externa:

- o salário médio de Engenheiro de Dados que a Silver estima difere **2,4%** do
  CAGED, registro administrativo oficial com 6.409 profissionais;
- os **60,6%** de gestores que apontam IA como prioridade convergem com os
  **67%** publicados pela Bain, que é co-autora da própria pesquisa.

---

## Divisão do grupo

| Pessoa | Entrega |
|---|---|
| Gusthavo | ingestão e camada Bronze |
| Alexandre | ETL, EDA e de-para entre edições |
| Allison | diagrama da arquitetura (Draw.io) e fluxo |
| Caio | camadas Silver e Gold, respostas das 7 perguntas, Power BI |

---

## Reprodutibilidade

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

python dev/tests/test_silver_local.py <pasta_com_os_csv>
```

Roda o **mesmo** `job_silver.py` que vai para o Glue, contra uma Bronze montada
a partir dos CSVs, sem consumir sessão do AWS Academy Lab. São **86 asserções**:
contagem por edição, unicidade da chave, `ano_pesquisa` conferido contra a data
de envio real, domínio dos booleanos, integridade dos rótulos entre edições e
contra-prova de cada grupo de múltipla escolha contra os binários da origem.

Última execução: **86/86**.

Passo a passo, incluindo os pré-requisitos do Windows, em
[`docs/COMO_RODAR_LOCAL.md`](docs/COMO_RODAR_LOCAL.md). O caminho do dado da
origem ao dashboard, e o que é versionado aqui, em
[`docs/ONDE_ESTAO_AS_BASES.md`](docs/ONDE_ESTAO_AS_BASES.md).

O mesmo código roda local e no Glue sem nenhum `if`: os caminhos vêm de variável
de ambiente.

```bash
TC3_PATH_BRONZE=... TC3_PATH_SILVER=... python glue/silver/job_silver.py
```

---

## Fonte

Pesquisa **State of Data Brasil**, comunidade Data Hackers em parceria com a
Bain & Company. Edições 2023-2024, 2024-2025 e 2025-2026.
<https://www.kaggle.com/datahackers/datasets>
