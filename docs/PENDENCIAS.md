# O que falta

Entregáveis, decisões fechadas e o que ainda depende de alguém. Ordem de
prioridade.

---

## Entregáveis

Os três estão entregues: material executivo em `apresentacao/`, diagrama da
arquitetura em `docs/arquitetura_tc3.drawio` e dentro do material, scripts e
notebooks em `glue/`, `sql/` e `notebooks/`.

## Em aberto

**Revisão do Alexandre.** Ele ficou de ler o material executivo. Combinado no
grupo em 07/09: a entrega espera essa leitura.

**Ensaio da apresentação.** São 25 slides para 10 a 15 minutos. Os slides 3, 16,
21 e 22 carregam o argumento sozinhos, então são eles que precisam de tempo.

## Decidido com o grupo em 07/09

**Qual desenho da arquitetura vai na apresentação.** Fica o nosso
(`docs/arquitetura_tc3.drawio`, com `.svg` e `.png` ao lado), que põe o Glue Data
Catalog como camada transversal às três camadas, que é como o crawler roda de
fato, e termina no Power BI, que foi a ferramenta usada. O do Gusthavo
(`docs/arquitetura_dw_gusthavo.png`) continua no repositório como registro da
versão anterior: ele mostrava o Catalog como etapa entre a Gold e o Athena e
terminava em QuickSight.

**Contagem de 2023.** O comentário `-- 5923 Respostas da pesquisa--` em
`sql/bronze/consulta_tb_dh_23.txt` estava errado. A base tem **5.293**,
conferido em [`VALIDACAO_BRONZE.md`](VALIDACAO_BRONZE.md) (5.293 linhas × 399
colunas, ids únicos batendo) e coerente com 5.293 + 5.215 + 3.494 = 14.002 da
Silver. Era transposição de dígito, já registrada em
[`DECISOES_E_ACHADOS.md`](DECISOES_E_ACHADOS.md). Comentário corrigido.

## Ajuste cosmético no catálogo

A tabela da Silver aparece no Glue Data Catalog como `state_data`, porque o
crawler nomeia pela última pasta do caminho (`silver/state_data/`). Destoa de
`bronze_dw_*` e `gold_dw_*`. Resolve com uma `CREATE VIEW` no Athena, ou
apontando o crawler para um caminho com o nome desejado.

## Git

O repositório do grupo é
[`GusthavoSoares/tech_challenge_data_analytics_fase_3`](https://github.com/GusthavoSoares/tech_challenge_data_analytics_fase_3).
O Pull Request #1, que levou para lá a Silver, a Gold, as respostas das sete
perguntas, o dashboard e o material executivo, foi aprovado e mesclado pelo
Gusthavo em 07/09. As duas histórias viraram uma só, então o que sai daqui já vai
direto no `main`, sem branch nem PR.

Fonte do desenho anterior da arquitetura, no Drive do Gusthavo:
<https://drive.google.com/file/d/1DngqFuv4RscUxi_CCq9VnvH3tNjJUTdW/view?usp=sharing>
