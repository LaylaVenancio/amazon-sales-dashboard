# amazon-sales-dashboard


1. [Contexto](#1-contexto)
2. [Documentação técnica](#2-documentação-técnica)
3. [Resultado para o negócio](#3-resultado-para-o-negócio)
4. [Como reproduzir](#4-como-reproduzir)

---

## 1. Contexto

**Pergunta de negócio:** onde o portfólio de nosso e-commerce está forte e onde há risco?

**Desafio de dados:** transformar dados de produtos, preços e avaliações em informação confiável para análise de portfólio, lidando com duas granularidades diferentes:

- Os dados financeiros (`discounted_price`, `actual_price`, `discount_percentage`) são **por produto** (`product_id`).
- As avaliações (`reviews`, `rating`, `rating_count`) são **compartilhadas entre produtos da mesma família**.

Somar o `rating_count` por produto infla o engajamento: são 23,8 milhões de avaliações contra 16.527.805 quando contadas uma vez por família. Toda a modelagem parte dessa decisão.

---

## 2. Documentação técnica

### Arquitetura medalhão

| Camada | Objetivo |
|---|---|
| **Bronze** | Dados originais, como extraídos da fonte (CSV hospedado no GitHub). |
| **Prata** | Dados tratados e confiáveis: definição de granularidade, limpeza e padronização. |
| **Ouro** | Dados prontos para análise e indicadores, em star schema. |

### Camada prata

- **Deduplicação.** Cada linha é uma leitura (web scraping) da página de um produto, e o parâmetro `qid` da URL mostra que todas ocorreram no mesmo dia, entre 05:15 e 13:00 UTC. Foi mantida apenas a última leitura de cada produto (1 linha por `product_id`), o que removeu 114 linhas (cerca de **7,78%**). Para a análise de portfólio, comparar o mesmo produto com horas de diferença traz pouco insumo analítico.
- **`product_family_id`.** Coluna artificial que agrupa os `product_id` que compartilham ao menos um `review_id`. Gerou **1.181 famílias**: 1.084 com um único produto e 97 com mais de um (267 produtos no total, no máximo 8 por família).
  - *Limitações:* cada produto traz no máximo 8 reviews, então variações com reviews totalmente distintas não são ligadas; e a família pode incluir modelos diferentes da mesma listagem (ex.: TP-Link UE300 e UE300C).
  - *Critério descartado:* igualdade de `rating` e `rating_count`, por gerar falsos positivos (coincidências entre produtos distintos) e falsos negativos (contagens que diferem em poucas unidades entre variações).
- **`rating_count_familia`.** Assume o maior `rating_count` entre os membros da família (em 18 famílias há diferença de 1 a 13 avaliações; a nota nunca diverge). Também foram criadas `tamanho_familia` e `qtd_reviews`, para auditoria.
- **Limpeza.** Remoção do símbolo ₹ dos preços, conversão do desconto de texto (`64%`) para percentual (0,64), conversão de `rating` e `rating_count` para número e padronização de texto (espaços nas extremidades e caracteres de controle). O `rating` inválido (`|`, em 1 produto) virou nulo, e não zero, para evitar distorções em cálculos futuros.

### Camada ouro: star schema

Como os dados têm duas granularidades, o modelo reúne tabelas fato em grãos diferentes que compartilham dimensões.

| Tabela | Tipo | Granularidade | Linhas |
|---|---|---|---|
| `f_produto` | Fato | 1 por `product_id` | 1.351 |
| `f_rating` | Fato | 1 por `product_family_id` | 1.181 |
| `f_review` | Fato | 1 por `review_id` | 9.266 |
| `d_familia` | Dimensão | 1 por `product_family_id` | 1.181 |
| `d_produto` | Dimensão | 1 por `product_id` | 1.351 |
| `d_categoria` | Dimensão | 1 por `product_id` | 1.351 |
| `d_avaliador` | Dimensão | 1 por `user_id` | 9.047 |

Em `f_review`, cada review pertence a uma única família, então reviews compartilhadas entre variações do mesmo produto aparecem uma única vez.

**Relacionamentos**

| Dimensão | Fato | Cardinalidade | Filtro |
|---|---|---|---|
| `d_produto[product_id]` | `f_produto` | 1:1 | Bidirecional |
| `d_categoria[product_id]` | `f_produto` | 1:1 | Bidirecional |
| `d_familia[product_family_id]` | `f_produto` | 1:N | Bidirecional |
| `d_familia[product_family_id]` | `f_rating` | 1:1 | Bidirecional |
| `d_familia[product_family_id]` | `f_review` | 1:N | Único |
| `d_avaliador[user_id]` | `f_review` | 1:N | Único |

**Decisões de modelagem**

- **Separação de granularidades.** `rating_count_familia` e `rating` ficam em uma tabela com uma linha por família, o que evita a inflação do engajamento. A nota média deve ser ponderada por `rating_count_familia`.
- **Categorias diferentes na mesma família.** Duas famílias reúnem produtos de categorias diferentes e entram nas duas na análise por categoria, então a soma por categoria pode superar levemente o total.
- **Valores nulos.** `rating` nulo em 1 família e `rating_count_familia` nulo em 2 famílias (todas com um único produto) foram mantidos como nulos, conforme definido na prata.
- **`qtd_reviews`.** Fica na prata para auditoria e não vai para a ouro, pois pode ser obtida pela contagem de `f_review` por família.

### Enriquecimento: análise de sentimento

Cada review recebe uma pontuação de sentimento calculada com o **VADER** (`vaderSentiment`), em Python, por script executado no Power Query. O resultado é incorporado à `f_review`.

- O texto analisado é a junção de `review_title` e `review_content`, pois o título costuma resumir a opinião do cliente. Quando só um deles existe, usa-se o disponível. Links são removidos antes da pontuação.
- `sentimento_score` vai de -1 (muito negativo) a +1 (muito positivo). `sentimento_classe` segue o VADER: **positivo** (score ≥ 0,05), **negativo** (score ≤ -0,05) e **neutro** (entre os dois).
- Duas classes extras evitam classificações incorretas: **sem texto** (título e conteúdo vazios) e **idioma não suportado** (mais da metade das letras fora do alfabeto latino, como reviews em hindi). Nesses casos o score é nulo, pois o VADER só entende inglês e devolveria 0, o que faria um review ilegível parecer neutro.

**Limitações**

- O VADER foi desenvolvido para inglês. Hinglish, gírias e sarcasmo tendem a ser classificados como neutros ou de forma imprecisa.
- Cada produto traz no máximo 8 reviews, e o `rating_count` chega a centenas de milhares. Os 9.266 reviews são uma pequena amostra e devem ser lidos como ilustrativos, não como representativos do total de avaliações.
- O dataset não traz nota individual por review (apenas a da família), então o sentimento não pode ser validado diretamente contra a nota de cada cliente.

---

## 3. Resultado para o negócio

Uma estrutura analítica bem construída permite entender o desempenho do portfólio, identificar oportunidades e apoiar decisões estratégicas com confiança.

### Indicadores

| Indicador | Valor |
|---|---|
| Produtos | 1.351 |
| Famílias de produtos | 1.181 |
| Avaliações (contadas uma vez por família) | 16,5 milhões |
| Nota média | 4,08 |
| Preço mediano | ₹ 899 |
| Desconto médio | 46,7% |
| Reviews positivos / neutros / negativos | 79,8% / 9,0% / 9,6% |

### Mensagens para a liderança

1. **Há risco onde o volume é maior.** 106 famílias têm muitas avaliações e nota abaixo da média (ex.: boAt Airdopes 121v2, nota 3,8 com 181 mil avaliações). O próximo passo é ler os reviews negativos para identificar a causa.
2. **Existem joias escondidas.** 124 famílias têm nota acima da média e pouca visibilidade, e merecem mais exposição.
3. **Desconto não compra satisfação.** A nota média cai de 4,24 (sem desconto) para 4,00 (descontos de 80% ou mais). É uma associação, não prova de causa.

Outros achados: 97% do portfólio está em três categorias (Electronics, Home&Kitchen e Computers&Accessories), e o sentimento dos reviews acompanha a nota (25,3% de negativos nos produtos de nota até 3,5, contra cerca de 4% nos de nota a partir de 4,6).

### Páginas do dashboard

1. **Visão Executiva:** indicadores gerais, concentração por categoria e destaques de risco e oportunidade.
2. **Produtos e Categorias:** desempenho por categoria e dispersão de famílias (volume de avaliações × nota).
3. **Customer Insights:** sentimento dos reviews por faixa de nota e por categoria.
4. **Preços e Estratégia Comercial:** distribuição de preços, descontos e relação entre desconto e nota.
5. **Recomendações:** padrões, oportunidades, riscos e mensagens para a liderança.

<!-- Adicione aqui capturas de tela das páginas, por exemplo: ![Visão Executiva](imagens/visao_executiva.png) -->

---

## 4. Como reproduzir

**Requisitos**

- Power BI Desktop
- Python configurado nas opções do Power BI Desktop, com os pacotes `pandas`, `matplotlib` e `vaderSentiment`:

```bash
pip install pandas matplotlib vaderSentiment
```

**Passos**

1. Baixe o dataset Amazon Sales (CSV hospedado neste repositório).
2. Abra o arquivo `.pbix` no Power BI Desktop e, se necessário, ajuste o caminho da fonte em *Transformar dados*.
3. Atualize o modelo. O script de sentimento só roda no Power BI Desktop: o serviço do Power BI não o executa sem um gateway pessoal.

---

**Tecnologias:** Power BI (Power Query/M e DAX), Python (`pandas`, `matplotlib`, `vaderSentiment`) · ETL · Data Quality · Data Modeling · Star Schema
