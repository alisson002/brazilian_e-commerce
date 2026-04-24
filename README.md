# Brazilian E-Commerce — Análise Exploratória de Dados (Olist)

Dataset público da Olist com **99.441 pedidos** realizados entre 2016 e 2018 no e-commerce brasileiro. Este projeto responde **27 perguntas analíticas** com Pandas e PostgreSQL, cobrindo receita, geografia, comportamento do consumidor, tendências temporais, segmentação de clientes (RFM), análise de vendedores, logística, reviews e cohort de retenção.

---

## Sobre os Dados

| Arquivo | Descrição | Registros |
|---------|-----------|-----------|
| `olist_orders_dataset.csv` | Tabela central de pedidos | 99.441 linhas, 8 colunas |
| `olist_order_items_dataset.csv` | Itens de cada pedido | 112.650 linhas, 7 colunas |
| `olist_order_payments_dataset.csv` | Pagamentos dos pedidos | 103.886 linhas, 5 colunas |
| `olist_order_reviews_dataset.csv` | Avaliações dos clientes | 99.224 linhas, 7 colunas |
| `olist_products_dataset.csv` | Catálogo de produtos | 32.951 linhas, 9 colunas |
| `olist_sellers_dataset.csv` | Dados dos vendedores | 3.095 linhas, 4 colunas |
| `olist_customers_dataset.csv` | Dados dos clientes | 99.441 linhas, 5 colunas |
| `olist_geolocation_dataset.csv` | Geolocalização por CEP | 1.000.163 linhas, 5 colunas |
| `product_category_name_translation.csv` | Tradução PT → EN das categorias | 71 linhas, 2 colunas |

**Fonte:** [Kaggle — Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

### Principais Variáveis

| Variável | Descrição | Tipo | Exemplos |
|----------|-----------|------|----------|
| `order_id` | ID único do pedido | str | hash MD5 |
| `order_status` | Status do pedido | str | delivered, canceled, shipped |
| `order_purchase_timestamp` | Data/hora da compra | datetime | 2017-10-02 10:56:33 |
| `payment_type` | Forma de pagamento | str | credit_card, boleto, voucher |
| `payment_value` | Valor pago | float | 99.33, 24.39 |
| `price` | Preço do item | float | 58.90, 239.90 |
| `freight_value` | Valor do frete | float | 13.29, 19.93 |
| `review_score` | Nota do cliente (1–5) | int | 1, 2, 3, 4, 5 |
| `product_category_name` | Categoria do produto (PT) | str | perfumaria, esporte_lazer |
| `customer_state` | Estado do cliente | str | SP, RJ, MG |
| `seller_state` | Estado do vendedor | str | SP, PR, MG |
| `customer_unique_id` | ID real do cliente (sem duplicatas por pedido) | str | hash MD5 |

---

## 15 Perguntas e Respostas

Cada pergunta é respondida com duas abordagens: **Pandas** (Python) e **PostgreSQL**.

---

### Pergunta 1: Qual o faturamento total da plataforma e o ticket médio por pedido?

**Objetivo:** Estabelecer a baseline financeira do negócio — receita total e valor típico de compra.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> payments = pd.read_csv("olist_order_payments_dataset.csv")
>
> resultado = (
>     payments.groupby('order_id').agg(
>         total_pago=('payment_value', 'sum')
>     )
> )
>
> print(f"Faturamento total:     R$ {resultado['total_pago'].sum():,.2f}")
> print(f"Ticket médio:          R$ {resultado['total_pago'].mean():.2f}")
> print(f"Mediana por pedido:    R$ {resultado['total_pago'].median():.2f}")
> print(f"Total de pedidos:      {len(resultado):,}")
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH pagamentos_por_pedido AS (
>     SELECT order_id,
>            SUM(payment_value) AS total_pago
>     FROM olist_order_payments_dataset
>     GROUP BY order_id
> )
> SELECT
>     ROUND(SUM(total_pago)::numeric, 2)        AS faturamento_total,
>     ROUND(AVG(total_pago)::numeric, 2)        AS ticket_medio,
>     ROUND(PERCENTILE_CONT(0.5)
>           WITHIN GROUP (ORDER BY total_pago)
>           ::numeric, 2)                        AS mediana,
>     COUNT(*)                                   AS total_pedidos
> FROM pagamentos_por_pedido;
> ```

</details>

**Insight:** A plataforma movimentou **R$ 16.008.872,12** em 99.440 pedidos. O ticket médio é de **R$ 160,99**, mas a mediana cai para **R$ 105,29** — a diferença indica que pedidos de alto valor puxam a média para cima. Estratégias de upsell e cross-sell têm espaço para atuar na faixa mediana.

---

### Pergunta 2: Quais as 10 categorias de produtos com maior receita?

**Objetivo:** Identificar as categorias que mais contribuem para o faturamento e definir prioridades de investimento.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items    = pd.read_csv("olist_order_items_dataset.csv")
> products = pd.read_csv("olist_products_dataset.csv")
> transl   = pd.read_csv("product_category_name_translation.csv")
>
> resultado = (
>     items
>     .merge(products[['product_id', 'product_category_name']], on='product_id', how='left')
>     .merge(transl, on='product_category_name', how='left')
>     .assign(categoria=lambda df: df['product_category_name_english']
>             .fillna(df['product_category_name']))
>     .groupby('categoria').agg(
>         receita=('price', 'sum'),
>         qtd_itens=('order_item_id', 'count'),
>         ticket_medio=('price', 'mean')
>     )
>     .sort_values('receita', ascending=False)
>     .head(10)
>     .round(2)
> )
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     COALESCE(t.product_category_name_english, p.product_category_name) AS categoria,
>     ROUND(SUM(i.price)::numeric, 2)   AS receita,
>     COUNT(*)                           AS qtd_itens,
>     ROUND(AVG(i.price)::numeric, 2)   AS ticket_medio
> FROM olist_order_items_dataset i
> LEFT JOIN olist_products_dataset p USING (product_id)
> LEFT JOIN product_category_name_translation t
>        ON p.product_category_name = t.product_category_name
> GROUP BY categoria
> ORDER BY receita DESC
> LIMIT 10;
> ```

</details>

**Insight:** **health_beauty** lidera com **R$ 1.258.681** em receita, seguida por **watches_gifts** (R$ 1.205.005) com o maior ticket médio do top 10 (R$ 201,14). As top 10 categorias respondem por aproximadamente **60% do faturamento total**. Categorias de alto ticket como watches_gifts e cool_stuff (R$ 167) merecem atenção especial em campanhas de performance.

---

### Pergunta 3: Como se distribuem os pagamentos por tipo e qual o valor médio de cada um?

**Objetivo:** Entender o comportamento financeiro dos clientes e o impacto das formas de pagamento no ticket médio.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> payments = pd.read_csv("olist_order_payments_dataset.csv")
>
> resultado = (
>     payments.groupby('payment_type').agg(
>         qtd_transacoes=('order_id', 'count'),
>         valor_total=('payment_value', 'sum'),
>         valor_medio=('payment_value', 'mean'),
>         media_parcelas=('payment_installments', 'mean')
>     )
>     .sort_values('qtd_transacoes', ascending=False)
>     .round(2)
> )
> resultado['participacao_pct'] = (
>     resultado['qtd_transacoes'] / resultado['qtd_transacoes'].sum() * 100
> ).round(1)
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     payment_type,
>     COUNT(*)                                          AS qtd_transacoes,
>     ROUND(SUM(payment_value)::numeric, 2)             AS valor_total,
>     ROUND(AVG(payment_value)::numeric, 2)             AS valor_medio,
>     ROUND(AVG(payment_installments)::numeric, 2)      AS media_parcelas,
>     ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()
>           ::numeric, 1)                               AS participacao_pct
> FROM olist_order_payments_dataset
> GROUP BY payment_type
> ORDER BY qtd_transacoes DESC;
> ```

</details>

**Insight:** O **cartão de crédito** domina com **73,9%** das transações (76.795) e média de **3,51 parcelas**, gerando R$ 12.542.084 — 78,3% do faturamento. O **boleto** representa 19% com ticket similar (R$ 145). O **voucher**, com apenas 5,6%, tem ticket médio de R$ 65,70 — provável uso em descontos. Oportunidade: incentivar parcelamento no boleto pode aumentar o ticket médio desse segmento.

---

### Pergunta 4: Quais os 10 estados com maior volume de pedidos e receita?

**Objetivo:** Mapear a concentração geográfica da demanda para direcionar esforços de marketing e logística.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv")
> customers = pd.read_csv("olist_customers_dataset.csv")
> payments  = pd.read_csv("olist_order_payments_dataset.csv")
>
> pay_por_pedido = payments.groupby('order_id')['payment_value'].sum().reset_index()
>
> resultado = (
>     orders
>     .merge(customers[['customer_id', 'customer_state']], on='customer_id', how='left')
>     .merge(pay_por_pedido, on='order_id', how='left')
>     .groupby('customer_state').agg(
>         qtd_pedidos=('order_id', 'count'),
>         receita_total=('payment_value', 'sum'),
>         ticket_medio=('payment_value', 'mean')
>     )
>     .sort_values('qtd_pedidos', ascending=False)
>     .head(10)
>     .round(2)
> )
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH pay_pedido AS (
>     SELECT order_id, SUM(payment_value) AS payment_value
>     FROM olist_order_payments_dataset
>     GROUP BY order_id
> )
> SELECT
>     c.customer_state,
>     COUNT(o.order_id)                             AS qtd_pedidos,
>     ROUND(SUM(p.payment_value)::numeric, 2)       AS receita_total,
>     ROUND(AVG(p.payment_value)::numeric, 2)       AS ticket_medio
> FROM olist_orders_dataset o
> LEFT JOIN olist_customers_dataset c USING (customer_id)
> LEFT JOIN pay_pedido p USING (order_id)
> GROUP BY c.customer_state
> ORDER BY qtd_pedidos DESC
> LIMIT 10;
> ```

</details>

**Insight:** **São Paulo** concentra **41.746 pedidos (42%)** e **R$ 5.998.226 (37%)** da receita. O trio SP+RJ+MG soma **66,3%** dos pedidos. **Bahia** e **Goiás** se destacam com tickets médios acima de R$ 170, sugerindo potencial de crescimento nessas regiões com menor penetração atual.

---

### Pergunta 5: Quais as 10 cidades com maior concentração de clientes?

**Objetivo:** Identificar os centros urbanos mais relevantes para campanhas georreferenciadas.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> customers = pd.read_csv("olist_customers_dataset.csv")
>
> resultado = (
>     customers.groupby(['customer_city', 'customer_state'])
>     .agg(qtd_clientes=('customer_unique_id', 'nunique'))
>     .sort_values('qtd_clientes', ascending=False)
>     .head(10)
>     .reset_index()
> )
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     customer_city,
>     customer_state,
>     COUNT(DISTINCT customer_unique_id) AS qtd_clientes
> FROM olist_customers_dataset
> GROUP BY customer_city, customer_state
> ORDER BY qtd_clientes DESC
> LIMIT 10;
> ```

</details>

**Insight:** **São Paulo** lidera com **14.984 clientes únicos** — mais que o dobro do 2º colocado, Rio de Janeiro (6.620). O top 10 inclui apenas cidades do Sul e Sudeste, exceto Salvador (BA) com 1.209 clientes. Guarulhos e São Bernardo do Campo reforçam a relevância da Grande São Paulo além da capital.

---

### Pergunta 6: Quais estados concentram mais vendedores e qual sua participação na receita?

**Objetivo:** Avaliar a distribuição geográfica da oferta e identificar dependência de fornecimento.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items   = pd.read_csv("olist_order_items_dataset.csv")
> sellers = pd.read_csv("olist_sellers_dataset.csv")
>
> resultado = (
>     items.merge(sellers[['seller_id', 'seller_state']], on='seller_id', how='left')
>     .groupby('seller_state').agg(
>         qtd_vendedores=('seller_id', 'nunique'),
>         receita=('price', 'sum')
>     )
>     .sort_values('receita', ascending=False)
>     .head(10)
>     .round(2)
> )
> resultado['pct_receita'] = (
>     resultado['receita'] / resultado['receita'].sum() * 100
> ).round(1)
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH receita_estado AS (
>     SELECT
>         s.seller_state,
>         COUNT(DISTINCT s.seller_id)       AS qtd_vendedores,
>         ROUND(SUM(i.price)::numeric, 2)   AS receita
>     FROM olist_order_items_dataset i
>     LEFT JOIN olist_sellers_dataset s USING (seller_id)
>     GROUP BY s.seller_state
> )
> SELECT
>     seller_state,
>     qtd_vendedores,
>     receita,
>     ROUND(receita * 100.0 / SUM(receita) OVER ()::numeric, 1) AS pct_receita
> FROM receita_estado
> ORDER BY receita DESC
> LIMIT 10;
> ```

</details>

**Insight:** **São Paulo** concentra **1.849 vendedores (59,7%)** e **64,4% da receita** gerada. Paraná é o 2º com 9,3%, seguido por MG (7,4%) e RJ (6,2%). Essa hiperpolarização cria risco: qualquer disrupção em SP afeta dois terços do negócio. Programas de onboarding em outros estados diversificariam a oferta.

---

### Pergunta 7: Como evoluiu o volume de pedidos mês a mês?

**Objetivo:** Identificar a trajetória de crescimento e possíveis sazonalidades na plataforma.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders = pd.read_csv("olist_orders_dataset.csv",
>                      parse_dates=['order_purchase_timestamp'])
>
> resultado = (
>     orders
>     .assign(ano_mes=orders['order_purchase_timestamp'].dt.to_period('M'))
>     .groupby('ano_mes').agg(qtd_pedidos=('order_id', 'count'))
>     .sort_index()
> )
> resultado['crescimento_pct'] = resultado['qtd_pedidos'].pct_change() * 100
> print(resultado.round(1).to_string())
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     TO_CHAR(DATE_TRUNC('month', order_purchase_timestamp), 'YYYY-MM') AS ano_mes,
>     COUNT(*) AS qtd_pedidos,
>     ROUND(
>         (COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY DATE_TRUNC('month', order_purchase_timestamp)))
>         * 100.0 / NULLIF(LAG(COUNT(*)) OVER (ORDER BY DATE_TRUNC('month', order_purchase_timestamp)), 0)
>         ::numeric, 1
>     ) AS crescimento_pct
> FROM olist_orders_dataset
> GROUP BY DATE_TRUNC('month', order_purchase_timestamp)
> ORDER BY ano_mes;
> ```

</details>

**Insight:** A plataforma saiu de **324 pedidos em outubro/2016** para **7.544 em novembro/2017** (Black Friday) — crescimento de **~23x em 2 anos**. O mês de novembro 2017 representa um salto de **+63%** sobre outubro, confirmando o impacto da Black Friday. Em 2018, os pedidos se estabilizaram entre **6.000–7.300/mês**, sinalizando maturidade operacional.

---

### Pergunta 8: Em quais dias da semana e horários os pedidos são mais frequentes?

**Objetivo:** Descobrir os melhores momentos para campanhas de e-mail marketing, notificações push e promoções.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders = pd.read_csv("olist_orders_dataset.csv",
>                      parse_dates=['order_purchase_timestamp'])
>
> orders['dia_semana'] = orders['order_purchase_timestamp'].dt.day_name()
> orders['hora']       = orders['order_purchase_timestamp'].dt.hour
>
> por_dia  = orders['dia_semana'].value_counts()
> por_hora = (
>     orders.groupby('hora')['order_id']
>     .count()
>     .sort_values(ascending=False)
>     .head(5)
> )
> print("Por dia da semana:\n", por_dia)
> print("\nTop 5 horários:\n", por_hora)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     TO_CHAR(order_purchase_timestamp, 'Day') AS dia_semana,
>     COUNT(*) AS qtd_pedidos
> FROM olist_orders_dataset
> GROUP BY TO_CHAR(order_purchase_timestamp, 'Day'),
>          EXTRACT(DOW FROM order_purchase_timestamp)
> ORDER BY EXTRACT(DOW FROM order_purchase_timestamp);
>
> SELECT
>     EXTRACT(HOUR FROM order_purchase_timestamp)::int AS hora,
>     COUNT(*) AS qtd_pedidos
> FROM olist_orders_dataset
> GROUP BY hora
> ORDER BY qtd_pedidos DESC
> LIMIT 5;
> ```

</details>

**Insight:** **Segunda-feira** lidera com **16.196 pedidos**, enquanto sábado registra apenas **10.887** — 33% menos. O pico de horário é às **16h (6.675 pedidos)**, com janela quente entre **11h e 16h**. Campanhas e e-mails disparados na segunda-feira entre 11h e 16h têm maior probabilidade de conversão.

---

### Pergunta 9: Como evoluiu a receita mensal e há sazonalidade?

**Objetivo:** Entender o comportamento da receita ao longo do tempo e identificar períodos de pico.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders   = pd.read_csv("olist_orders_dataset.csv",
>                        parse_dates=['order_purchase_timestamp'])
> payments = pd.read_csv("olist_order_payments_dataset.csv")
>
> pay_por_pedido = payments.groupby('order_id')['payment_value'].sum().reset_index()
>
> resultado = (
>     orders.merge(pay_por_pedido, on='order_id', how='left')
>     .assign(ano_mes=lambda df: df['order_purchase_timestamp'].dt.to_period('M'))
>     .groupby('ano_mes').agg(
>         receita=('payment_value', 'sum'),
>         qtd_pedidos=('order_id', 'count')
>     )
>     .sort_index()
>     .round(2)
> )
> print(resultado.to_string())
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH pay_pedido AS (
>     SELECT order_id, SUM(payment_value) AS payment_value
>     FROM olist_order_payments_dataset
>     GROUP BY order_id
> )
> SELECT
>     TO_CHAR(DATE_TRUNC('month', o.order_purchase_timestamp), 'YYYY-MM') AS ano_mes,
>     ROUND(SUM(p.payment_value)::numeric, 2) AS receita,
>     COUNT(o.order_id)                        AS qtd_pedidos,
>     ROUND(AVG(p.payment_value)::numeric, 2)  AS ticket_medio
> FROM olist_orders_dataset o
> LEFT JOIN pay_pedido p USING (order_id)
> GROUP BY DATE_TRUNC('month', o.order_purchase_timestamp)
> ORDER BY ano_mes;
> ```

</details>

**Insight:** O pico de receita foi em **novembro/2017** com **R$ 1.194.882** (Black Friday), representando um salto de **+77%** sobre agosto/2017 (R$ 674.396). Em 2018, a receita mensal média ficou em **R$ 1.090.000**, com variação controlada — sinal de maturidade. O crescimento acumulado de ago/2017 a ago/2018 foi de **+51,5%**.

---

### Pergunta 10: Qual a distribuição dos status dos pedidos?

**Objetivo:** Avaliar a saúde operacional da plataforma — taxa de entrega, cancelamentos e pedidos problemáticos.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders = pd.read_csv("olist_orders_dataset.csv")
>
> contagem = orders['order_status'].value_counts()
> resultado = pd.DataFrame({
>     'qtd': contagem,
>     'pct': (contagem / len(orders) * 100).round(2)
> })
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     order_status,
>     COUNT(*) AS qtd,
>     ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()::numeric, 2) AS pct
> FROM olist_orders_dataset
> GROUP BY order_status
> ORDER BY qtd DESC;
> ```

</details>

**Insight:** **97,02%** dos pedidos (96.478) foram entregues com sucesso — excelente taxa operacional. Cancelamentos representam apenas **0,63%** (625 pedidos). Pedidos `unavailable` (609) e `invoiced` (314) somam 923 casos que não avançaram no fluxo e merecem investigação. `shipped` (1.107) provavelmente estavam em trânsito no momento do corte do dataset.

---

### Pergunta 11: Qual a taxa de recompra dos clientes?

**Objetivo:** Medir a fidelidade da base de clientes e o potencial de receita recorrente.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv")
> customers = pd.read_csv("olist_customers_dataset.csv")
>
> compras_por_cliente = (
>     orders
>     .merge(customers[['customer_id', 'customer_unique_id']], on='customer_id', how='left')
>     .groupby('customer_unique_id')['order_id']
>     .count()
> )
>
> resultado = (
>     compras_por_cliente
>     .value_counts()
>     .rename_axis('n_pedidos')
>     .reset_index(name='qtd_clientes')
> )
> resultado['pct'] = (resultado['qtd_clientes'] / resultado['qtd_clientes'].sum() * 100).round(2)
>
> recompra = (compras_por_cliente > 1).sum()
> total    = len(compras_por_cliente)
> print(resultado)
> print(f"\nRecompra: {recompra} clientes ({recompra/total*100:.2f}%)")
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH pedidos_por_cliente AS (
>     SELECT
>         c.customer_unique_id,
>         COUNT(o.order_id) AS n_pedidos
>     FROM olist_orders_dataset o
>     LEFT JOIN olist_customers_dataset c USING (customer_id)
>     GROUP BY c.customer_unique_id
> )
> SELECT
>     n_pedidos,
>     COUNT(*)                                                      AS qtd_clientes,
>     ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()::numeric, 2)  AS pct
> FROM pedidos_por_cliente
> GROUP BY n_pedidos
> ORDER BY n_pedidos;
> ```

</details>

**Insight:** **96,88%** dos clientes (93.099 de 96.096) fizeram apenas **1 compra**. Apenas **3,12%** (2.997 clientes) recompraram — e desses, 2.745 fizeram exatamente 2 pedidos. Um cliente chegou a **17 pedidos**. A baixa taxa de recompra sinaliza enorme oportunidade para programas de fidelidade, remarketing e automação de e-mail pós-compra.

---

### Pergunta 12: Qual o prazo médio de entrega real versus estimado por estado?

**Objetivo:** Identificar onde a promessa de entrega é cumprida e onde há gap entre estimativa e realidade.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv", parse_dates=[
>     'order_purchase_timestamp', 'order_delivered_customer_date',
>     'order_estimated_delivery_date'])
> customers = pd.read_csv("olist_customers_dataset.csv")
>
> delivered = orders[orders['order_status'] == 'delivered'].copy()
> delivered['dias_reais']     = (delivered['order_delivered_customer_date']
>                                - delivered['order_purchase_timestamp']).dt.days
> delivered['dias_estimados'] = (delivered['order_estimated_delivery_date']
>                                - delivered['order_purchase_timestamp']).dt.days
> delivered['antecedencia']   = delivered['dias_estimados'] - delivered['dias_reais']
>
> resultado = (
>     delivered
>     .merge(customers[['customer_id', 'customer_state']], on='customer_id', how='left')
>     .groupby('customer_state').agg(
>         media_real=('dias_reais', 'mean'),
>         media_estimado=('dias_estimados', 'mean'),
>         antecedencia_media=('antecedencia', 'mean'),
>         qtd_pedidos=('order_id', 'count')
>     )
>     .sort_values('media_real')
>     .round(1)
> )
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     c.customer_state,
>     ROUND(AVG(
>         o.order_delivered_customer_date::date - o.order_purchase_timestamp::date
>     )::numeric, 1) AS media_real,
>     ROUND(AVG(
>         o.order_estimated_delivery_date::date - o.order_purchase_timestamp::date
>     )::numeric, 1) AS media_estimado,
>     ROUND(AVG(
>         (o.order_estimated_delivery_date::date - o.order_purchase_timestamp::date)
>       - (o.order_delivered_customer_date::date - o.order_purchase_timestamp::date)
>     )::numeric, 1) AS antecedencia_media,
>     COUNT(*)        AS qtd_pedidos
> FROM olist_orders_dataset o
> LEFT JOIN olist_customers_dataset c USING (customer_id)
> WHERE o.order_status = 'delivered'
> GROUP BY c.customer_state
> ORDER BY media_real;
> ```

</details>

**Insight:** A média geral de entrega é **12,1 dias reais** contra **23,4 dias estimados** — os pedidos chegam em média **11,3 dias antes** do prazo. **SP** é o mais rápido (**8,3 dias**) por concentrar mais vendedores e hubs logísticos. **Roraima (RR)** leva **29 dias** — 3,5x mais que SP. Estados do Norte (RO, AC, AM, AP) têm estimativas muito conservadoras, com antecedência de 18–20 dias — há espaço para ajustar o prazo prometido e melhorar a experiência.

---

### Pergunta 13: Existe correlação entre a nota de avaliação e o prazo de entrega?

**Objetivo:** Quantificar o impacto da experiência de entrega na satisfação do cliente.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders  = pd.read_csv("olist_orders_dataset.csv", parse_dates=[
>     'order_purchase_timestamp', 'order_delivered_customer_date',
>     'order_estimated_delivery_date'])
> reviews = pd.read_csv("olist_order_reviews_dataset.csv")
>
> delivered = orders[orders['order_status'] == 'delivered'].copy()
> delivered['dias_reais']   = (delivered['order_delivered_customer_date']
>                              - delivered['order_purchase_timestamp']).dt.days
> delivered['antecedencia'] = (delivered['order_estimated_delivery_date']
>                              - delivered['order_delivered_customer_date']).dt.days
>
> df = delivered.merge(reviews[['order_id', 'review_score']], on='order_id', how='inner')
>
> resultado = (
>     df.groupby('review_score').agg(
>         media_dias_reais=('dias_reais', 'mean'),
>         media_antecedencia=('antecedencia', 'mean'),
>         qtd=('order_id', 'count')
>     )
>     .round(1)
> )
> corr = df[['review_score', 'dias_reais']].corr().iloc[0, 1]
> print(resultado)
> print(f"\nCorrelação review_score x dias_reais: {corr:.3f}")
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH metricas AS (
>     SELECT
>         r.review_score,
>         o.order_delivered_customer_date::date - o.order_purchase_timestamp::date
>             AS dias_reais,
>         o.order_estimated_delivery_date::date - o.order_delivered_customer_date::date
>             AS antecedencia
>     FROM olist_orders_dataset o
>     INNER JOIN olist_order_reviews_dataset r USING (order_id)
>     WHERE o.order_status = 'delivered'
> )
> SELECT
>     review_score,
>     ROUND(AVG(dias_reais)::numeric, 1)    AS media_dias_reais,
>     ROUND(AVG(antecedencia)::numeric, 1)  AS media_antecedencia,
>     COUNT(*)                               AS qtd,
>     ROUND(CORR(review_score, dias_reais)::numeric, 3) AS correlacao
> FROM metricas
> GROUP BY review_score
> ORDER BY review_score;
> ```

</details>

**Insight:** A correlação entre nota e prazo é **-0,334** — moderada e negativa: quanto mais dias o cliente espera, menor a nota. Clientes com nota **1 esperaram 20,8 dias** em média; com nota **5, apenas 10,2 dias**. A antecedência (dias antes do prazo prometido) vai de **3,4 dias** (nota 1) a **12,8 dias** (nota 5). Isso revela que **cumprir o prazo prometido importa mais que a velocidade absoluta** — ajustar estimativas para baixo pode aumentar satisfação sem mudar a logística.

---

### Pergunta 14: Qual a correlação entre frete, preço do produto e peso?

**Objetivo:** Entender os drivers do custo de frete para subsidiar decisões de precificação e negociação logística.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items    = pd.read_csv("olist_order_items_dataset.csv")
> products = pd.read_csv("olist_products_dataset.csv")
>
> df = items.merge(products[['product_id', 'product_weight_g']], on='product_id', how='left')
>
> corr = df[['price', 'freight_value', 'product_weight_g']].corr().round(3)
> print("Matriz de correlação:\n", corr)
>
> frete_faixa = pd.cut(df['price'],
>                      bins=[0, 50, 200, 500, 10000],
>                      labels=['Até R$50', 'R$50–200', 'R$200–500', 'Acima R$500'])
> frete_medio = df.groupby(frete_faixa, observed=True)['freight_value'].mean().round(2)
> print("\nFrete médio por faixa de preço:\n", frete_medio)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     ROUND(CORR(i.freight_value, p.product_weight_g)::numeric, 3) AS corr_frete_peso,
>     ROUND(CORR(i.freight_value, i.price)::numeric, 3)             AS corr_frete_preco
> FROM olist_order_items_dataset i
> LEFT JOIN olist_products_dataset p USING (product_id);
>
> SELECT
>     CASE
>         WHEN i.price <= 50   THEN 'Até R$50'
>         WHEN i.price <= 200  THEN 'R$50–200'
>         WHEN i.price <= 500  THEN 'R$200–500'
>         ELSE 'Acima R$500'
>     END AS faixa_preco,
>     ROUND(AVG(i.freight_value)::numeric, 2) AS frete_medio,
>     COUNT(*)                                 AS qtd_itens
> FROM olist_order_items_dataset i
> GROUP BY faixa_preco
> ORDER BY frete_medio;
> ```

</details>

**Insight:** O **peso** é o principal driver do frete com correlação de **0,610** — bem acima da correlação com preço (**0,414**). Produtos acima de R$ 500 têm frete médio de **R$ 47,54**, mais que o triplo dos produtos até R$ 50 (**R$ 14,77**). Isso penaliza vendedores de produtos pesados e baratos, onde o frete pode representar 20–30% do preço final. Política de frete grátis acima de determinado valor de compra pode incentivar upsell e compensar esse desequilíbrio.

---

### Pergunta 15: Como se segmentam os clientes pelo modelo RFM?

**Objetivo:** Classificar a base de clientes em grupos estratégicos para campanhas diferenciadas de retenção e reativação.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv", parse_dates=['order_purchase_timestamp'])
> customers = pd.read_csv("olist_customers_dataset.csv")
> payments  = pd.read_csv("olist_order_payments_dataset.csv")
>
> snapshot_date  = orders['order_purchase_timestamp'].max() + pd.Timedelta(days=1)
> pay_por_pedido = payments.groupby('order_id')['payment_value'].sum().reset_index()
>
> base = (
>     orders[orders['order_status'] == 'delivered']
>     .merge(customers[['customer_id', 'customer_unique_id']], on='customer_id', how='left')
>     .merge(pay_por_pedido, on='order_id', how='left')
> )
>
> rfm = base.groupby('customer_unique_id').agg(
>     recencia=('order_purchase_timestamp', lambda x: (snapshot_date - x.max()).days),
>     frequencia=('order_id', 'count'),
>     monetario=('payment_value', 'sum')
> ).round(2)
>
> rfm['R'] = pd.qcut(rfm['recencia'],  5, labels=[5,4,3,2,1]).astype(int)
> rfm['F'] = pd.qcut(rfm['frequencia'].rank(method='first'), 5, labels=[1,2,3,4,5]).astype(int)
> rfm['M'] = pd.qcut(rfm['monetario'], 5, labels=[1,2,3,4,5]).astype(int)
> rfm['rfm_score'] = rfm['R'] + rfm['F'] + rfm['M']
>
> def segmento(s):
>     if s >= 13:  return 'Champions'
>     elif s >= 10: return 'Loyal Customers'
>     elif s >= 7:  return 'Potential Loyalists'
>     elif s >= 5:  return 'At Risk'
>     else:         return 'Lost'
>
> rfm['segmento'] = rfm['rfm_score'].apply(segmento)
>
> resultado = (
>     rfm.groupby('segmento').agg(
>         qtd_clientes=('rfm_score', 'count'),
>         recencia_media=('recencia', 'mean'),
>         frequencia_media=('frequencia', 'mean'),
>         valor_medio=('monetario', 'mean')
>     )
>     .round(1)
>     .sort_values('qtd_clientes', ascending=False)
> )
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH snapshot AS (
>     SELECT MAX(order_purchase_timestamp) + INTERVAL '1 day' AS snapshot_date
>     FROM olist_orders_dataset
> ),
> rfm_base AS (
>     SELECT
>         c.customer_unique_id,
>         DATE_PART('day', s.snapshot_date - MAX(o.order_purchase_timestamp)) AS recencia,
>         COUNT(DISTINCT o.order_id)                                           AS frequencia,
>         SUM(p.payment_value)                                                 AS monetario
>     FROM olist_orders_dataset o
>     JOIN olist_customers_dataset c USING (customer_id)
>     JOIN olist_order_payments_dataset p USING (order_id)
>     CROSS JOIN snapshot s
>     WHERE o.order_status = 'delivered'
>     GROUP BY c.customer_unique_id, s.snapshot_date
> ),
> rfm_scores AS (
>     SELECT *,
>         NTILE(5) OVER (ORDER BY recencia DESC)  AS r_score,
>         NTILE(5) OVER (ORDER BY frequencia ASC) AS f_score,
>         NTILE(5) OVER (ORDER BY monetario ASC)  AS m_score
>     FROM rfm_base
> )
> SELECT
>     CASE
>         WHEN r_score + f_score + m_score >= 13 THEN 'Champions'
>         WHEN r_score + f_score + m_score >= 10 THEN 'Loyal Customers'
>         WHEN r_score + f_score + m_score >= 7  THEN 'Potential Loyalists'
>         WHEN r_score + f_score + m_score >= 5  THEN 'At Risk'
>         ELSE                                        'Lost'
>     END AS segmento,
>     COUNT(*)                           AS qtd_clientes,
>     ROUND(AVG(recencia)::numeric, 1)   AS recencia_media,
>     ROUND(AVG(frequencia)::numeric, 1) AS frequencia_media,
>     ROUND(AVG(monetario)::numeric, 1)  AS valor_medio
> FROM rfm_scores
> GROUP BY segmento
> ORDER BY qtd_clientes DESC;
> ```

</details>

**Insight:** Os **93.358 clientes** segmentados revelam: **Potential Loyalists** são o maior grupo (**38.399 — 41,1%**) com gasto médio de R$ 129 e recência de 314 dias — alvo ideal para campanhas de reativação. **Loyal Customers** (31.503 — 33,7%) gastam R$ 214 em média. **Champions** (8.016 — 8,6%) são os melhores: compra recente (**142 dias**) e ticket de **R$ 334**. Os **Lost** (3.164 — 3,4%) não compram há ~486 dias — candidatos a win-back com cupons agressivos. Converter apenas 10% dos Potential Loyalists em Champions representaria +R$ 800K de receita adicional.

---

---

## Análise de Vendedores

### Pergunta 16: Como se distribui a receita entre os vendedores? (Curva de Pareto)

**Objetivo:** Identificar a concentração de receita nos vendedores e o risco de dependência de poucos fornecedores.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items = pd.read_csv("olist_order_items_dataset.csv")
>
> seller_rev = (
>     items.groupby('seller_id').agg(
>         receita=('price', 'sum'),
>         qtd_pedidos=('order_id', 'nunique'),
>         qtd_itens=('order_item_id', 'count')
>     )
>     .sort_values('receita', ascending=False)
>     .round(2)
> )
>
> seller_rev['receita_acum_pct'] = (
>     seller_rev['receita'].cumsum() / seller_rev['receita'].sum() * 100
> ).round(2)
>
> total = len(seller_rev)
> n_80  = (seller_rev['receita_acum_pct'] <= 80).sum()
>
> print(f"Total de vendedores: {total}")
> print(f"Vendedores que geram 80% da receita: {n_80} ({n_80/total*100:.1f}%)")
> print(f"\nTop 10 vendedores por receita:")
> print(seller_rev.head(10))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH receita_seller AS (
>     SELECT
>         seller_id,
>         ROUND(SUM(price)::numeric, 2)              AS receita,
>         COUNT(DISTINCT order_id)                   AS qtd_pedidos,
>         SUM(SUM(price)) OVER ()                    AS receita_total
>     FROM olist_order_items_dataset
>     GROUP BY seller_id
> ),
> pareto AS (
>     SELECT *,
>         ROUND(SUM(receita) OVER (ORDER BY receita DESC)
>               / receita_total * 100::numeric, 2)   AS receita_acum_pct
>     FROM receita_seller
> )
> SELECT seller_id, receita, qtd_pedidos, receita_acum_pct
> FROM pareto
> ORDER BY receita DESC
> LIMIT 20;
> ```

</details>

**Insight:** Apenas **543 vendedores (17,5% dos 3.095)** geram **80% da receita** — clássica distribuição de Pareto. O top 10 já concentra **13,1% do faturamento** (R$ 1.787.241). O maior vendedor sozinho gerou **R$ 229.472** com 1.132 pedidos. Essa concentração representa risco operacional: a saída dos 50 maiores vendedores afetaria ~25% da receita da plataforma.

---

### Pergunta 17: Quais vendedores têm as piores e melhores avaliações médias dos clientes?

**Objetivo:** Identificar vendedores que comprometem a reputação da plataforma e os que se destacam positivamente.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items   = pd.read_csv("olist_order_items_dataset.csv")
> orders  = pd.read_csv("olist_orders_dataset.csv")
> reviews = pd.read_csv("olist_order_reviews_dataset.csv")
>
> seller_review = (
>     items[['order_id', 'seller_id', 'price']]
>     .merge(reviews[['order_id', 'review_score']], on='order_id', how='left')
>     .groupby('seller_id').agg(
>         media_score=('review_score', 'mean'),
>         qtd_avaliacoes=('review_score', 'count'),
>         receita=('price', 'sum')
>     )
>     .round(2)
> )
>
> # Filtra vendedores com volume mínimo de avaliações
> resultado = seller_review[seller_review['qtd_avaliacoes'] >= 30]
> print(f"Vendedores com >= 30 avaliações: {len(resultado)}")
> print(f"\nScore médio da plataforma: {resultado['media_score'].mean():.2f}")
> print(f"\nPiores 10 vendedores:")
> print(resultado.sort_values('media_score').head(10))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     i.seller_id,
>     ROUND(AVG(r.review_score)::numeric, 2) AS media_score,
>     COUNT(r.review_score)                  AS qtd_avaliacoes,
>     ROUND(SUM(i.price)::numeric, 2)        AS receita
> FROM olist_order_items_dataset i
> LEFT JOIN olist_order_reviews_dataset r USING (order_id)
> GROUP BY i.seller_id
> HAVING COUNT(r.review_score) >= 30
> ORDER BY media_score ASC
> LIMIT 10;
> ```

</details>

**Insight:** Entre os **684 vendedores** com 30+ avaliações, a média da plataforma é **4,07**. O pior vendedor tem média de **2,20** com 136 avaliações — e ainda assim gera R$ 13.341 de receita, indicando que clientes insatisfeitos continuam comprando (talvez sem alternativas). A distribuição é assimétrica: **75% dos vendedores** têm score acima de 3,89, mas o mínimo é 2,20. Cruzar score com receita permite criar uma matriz risco × valor para priorizar intervenções.

---

## Análise Logística

### Pergunta 18: Quais pedidos chegaram após o prazo estimado e onde isso ocorre mais?

**Objetivo:** Quantificar falhas de entrega e identificar estados e categorias com maior taxa de atraso.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv", parse_dates=[
>     'order_purchase_timestamp', 'order_delivered_customer_date',
>     'order_estimated_delivery_date'])
> customers = pd.read_csv("olist_customers_dataset.csv")
>
> delivered = orders[orders['order_status'] == 'delivered'].copy()
> delivered['atrasado']   = delivered['order_delivered_customer_date'] > delivered['order_estimated_delivery_date']
> delivered['dias_atraso'] = (
>     delivered['order_delivered_customer_date'] - delivered['order_estimated_delivery_date']
> ).dt.days.clip(lower=0)
>
> total = len(delivered)
> atrasados = delivered['atrasado'].sum()
> print(f"Entregas com atraso: {atrasados} ({atrasados/total*100:.2f}%)")
> print(f"Atraso médio (quando atrasado): {delivered[delivered['atrasado']]['dias_atraso'].mean():.1f} dias")
>
> resultado = (
>     delivered
>     .merge(customers[['customer_id', 'customer_state']], on='customer_id', how='left')
>     .groupby('customer_state').agg(
>         total=('order_id', 'count'),
>         atrasados=('atrasado', 'sum'),
>         atraso_medio=('dias_atraso', 'mean')
>     )
>     .assign(pct_atraso=lambda df: (df['atrasados'] / df['total'] * 100).round(1))
>     .sort_values('pct_atraso', ascending=False)
>     .round(2)
> )
> print(resultado.head(10))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     c.customer_state,
>     COUNT(*)                                               AS total,
>     SUM(CASE WHEN o.order_delivered_customer_date
>                   > o.order_estimated_delivery_date THEN 1 ELSE 0 END) AS atrasados,
>     ROUND(SUM(CASE WHEN o.order_delivered_customer_date
>                        > o.order_estimated_delivery_date THEN 1 ELSE 0 END)
>           * 100.0 / COUNT(*)::numeric, 1)                 AS pct_atraso,
>     ROUND(AVG(GREATEST(
>         o.order_delivered_customer_date::date - o.order_estimated_delivery_date::date,
>         0))::numeric, 1)                                   AS atraso_medio_dias
> FROM olist_orders_dataset o
> LEFT JOIN olist_customers_dataset c USING (customer_id)
> WHERE o.order_status = 'delivered'
> GROUP BY c.customer_state
> ORDER BY pct_atraso DESC;
> ```

</details>

**Insight:** **8,11%** dos pedidos entregues (7.826 de 96.478) chegaram após o prazo estimado — com atraso médio de **8,9 dias** e máximo de **188 dias**. Os estados mais afetados são **Alagoas (23,9%)**, **Maranhão (19,7%)** e **Piauí (16,0%)**. Por categoria, **áudio (13,1%)** e **moda praia (12,8%)** lideram. Interessante: o Norte tem alta taxa de atraso mas estimativas conservadoras (Q12), enquanto o Nordeste parece ter estimativas que não cobrem a realidade logística local.

---

### Pergunta 19: Quanto tempo os vendedores levam para enviar o pedido ao transportador após a aprovação?

**Objetivo:** Identificar gargalos no processo do vendedor que impactam o prazo total de entrega.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders  = pd.read_csv("olist_orders_dataset.csv", parse_dates=[
>     'order_approved_at', 'order_delivered_carrier_date'])
> items   = pd.read_csv("olist_order_items_dataset.csv")
> sellers = pd.read_csv("olist_sellers_dataset.csv")
>
> shipped = orders.dropna(subset=['order_approved_at', 'order_delivered_carrier_date']).copy()
> shipped['horas_preparo'] = (
>     shipped['order_delivered_carrier_date'] - shipped['order_approved_at']
> ).dt.total_seconds() / 3600
> shipped = shipped[shipped['horas_preparo'] >= 0]
>
> seller_estado = (items[['order_id', 'seller_id']].drop_duplicates('order_id')
>                  .merge(sellers[['seller_id', 'seller_state']], on='seller_id', how='left'))
>
> resultado = (
>     shipped.merge(seller_estado, on='order_id', how='left')
>     .groupby('seller_state').agg(
>         media_horas=('horas_preparo', 'mean'),
>         mediana_horas=('horas_preparo', 'median'),
>         qtd=('order_id', 'count')
>     )
>     .sort_values('media_horas')
>     .round(1)
> )
> print(f"Média geral: {shipped['horas_preparo'].mean():.1f}h ({shipped['horas_preparo'].mean()/24:.1f} dias)")
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     s.seller_state,
>     ROUND(AVG(EXTRACT(EPOCH FROM
>         (o.order_delivered_carrier_date - o.order_approved_at)) / 3600)::numeric, 1)
>         AS media_horas,
>     ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
>         ORDER BY EXTRACT(EPOCH FROM
>             (o.order_delivered_carrier_date - o.order_approved_at)) / 3600)
>         ::numeric, 1)
>         AS mediana_horas,
>     COUNT(*) AS qtd
> FROM olist_orders_dataset o
> JOIN olist_order_items_dataset i USING (order_id)
> JOIN olist_sellers_dataset s USING (seller_id)
> WHERE o.order_approved_at IS NOT NULL
>   AND o.order_delivered_carrier_date IS NOT NULL
>   AND o.order_delivered_carrier_date >= o.order_approved_at
> GROUP BY s.seller_state
> ORDER BY media_horas;
> ```

</details>

**Insight:** O tempo médio de preparo (aprovação → envio ao transportador) é de **68,6 horas (2,9 dias)**, com mediana de **44,4h**. Vendedores do **Piauí** são os mais rápidos (35,9h), enquanto os do **Maranhão** são os mais lentos (113,3h — mais de 4,5 dias). SP, que concentra 68K pedidos, tem média de 68,6h — exatamente na média nacional. Reduzir o tempo de preparo dos 20% mais lentos para a mediana nacional economizaria ~1 dia no prazo final de entrega para milhares de pedidos.

---

## Análise de Reviews

### Pergunta 20: Quais categorias têm as melhores e piores avaliações médias dos clientes?

**Objetivo:** Identificar categorias problemáticas que corroem a satisfação e as que consistentemente encantam.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items    = pd.read_csv("olist_order_items_dataset.csv")
> products = pd.read_csv("olist_products_dataset.csv")
> transl   = pd.read_csv("product_category_name_translation.csv")
> reviews  = pd.read_csv("olist_order_reviews_dataset.csv")
>
> cat_reviews = (
>     items
>     .merge(products[['product_id', 'product_category_name']], on='product_id', how='left')
>     .merge(transl, on='product_category_name', how='left')
>     .assign(categoria=lambda df: df['product_category_name_english']
>             .fillna(df['product_category_name']))
>     [['order_id', 'categoria']].drop_duplicates('order_id')
>     .merge(reviews[['order_id', 'review_score']], on='order_id', how='inner')
>     .groupby('categoria').agg(
>         media_score=('review_score', 'mean'),
>         pct_negativas=('review_score', lambda x: (x <= 2).sum() / len(x) * 100),
>         pct_cinco=('review_score', lambda x: (x == 5).sum() / len(x) * 100),
>         qtd=('review_score', 'count')
>     )
>     .round(2)
> )
>
> resultado = cat_reviews[cat_reviews['qtd'] >= 100]
> print("Piores 10:")
> print(resultado.sort_values('media_score').head(10))
> print("\nMelhores 10:")
> print(resultado.sort_values('media_score', ascending=False).head(10))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     COALESCE(t.product_category_name_english, p.product_category_name) AS categoria,
>     ROUND(AVG(r.review_score)::numeric, 2)                              AS media_score,
>     ROUND(SUM(CASE WHEN r.review_score <= 2 THEN 1 ELSE 0 END)
>           * 100.0 / COUNT(*)::numeric, 1)                               AS pct_negativas,
>     ROUND(SUM(CASE WHEN r.review_score = 5 THEN 1 ELSE 0 END)
>           * 100.0 / COUNT(*)::numeric, 1)                               AS pct_cinco,
>     COUNT(*)                                                             AS qtd
> FROM olist_order_items_dataset i
> LEFT JOIN olist_products_dataset p USING (product_id)
> LEFT JOIN product_category_name_translation t ON p.product_category_name = t.product_category_name
> INNER JOIN olist_order_reviews_dataset r USING (order_id)
> GROUP BY categoria
> HAVING COUNT(*) >= 100
> ORDER BY media_score ASC
> LIMIT 10;
> ```

</details>

**Insight:** **office_furniture** é a categoria mais problemática com score médio de **3,62** e **22,6% de avaliações negativas** (notas 1–2). **fashion_male_clothing** tem a maior taxa de notas 1–2 (**26,1%**). No extremo oposto, **books_general_interest** lidera com **4,47** de média e **73,3% de notas 5**. A dicotomia é clara: **produtos físicos com entrega complexa** (móveis, eletros) têm piores avaliações; **produtos leves e com expectativas bem definidas** (livros, alimentos) têm as melhores.

---

### Pergunta 21: Quanto tempo a plataforma leva para responder às avaliações dos clientes?

**Objetivo:** Avaliar a agilidade no tratamento do feedback do cliente e se o score da avaliação influencia a velocidade de resposta.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> reviews = pd.read_csv("olist_order_reviews_dataset.csv",
>                       parse_dates=['review_creation_date', 'review_answer_timestamp'])
>
> reviews['horas_resposta'] = (
>     reviews['review_answer_timestamp'] - reviews['review_creation_date']
> ).dt.total_seconds() / 3600
>
> validos = reviews[reviews['horas_resposta'] >= 0]
>
> print(f"Tempo médio de resposta: {validos['horas_resposta'].mean():.1f}h "
>       f"({validos['horas_resposta'].mean()/24:.1f} dias)")
> print(f"Mediana: {validos['horas_resposta'].median():.1f}h")
>
> por_score = (
>     validos.groupby('review_score')['horas_resposta']
>     .agg(['mean', 'median', 'count'])
>     .round(1)
> )
> print("\nTempo de resposta por nota:")
> print(por_score)
>
> faixas = pd.cut(validos['horas_resposta'],
>                 bins=[0, 24, 72, 168, 720, 99999],
>                 labels=['<1 dia', '1-3 dias', '3-7 dias', '7-30 dias', '>30 dias'])
> print("\nDistribuição dos tempos:")
> print((faixas.value_counts(normalize=True) * 100).round(1))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     review_score,
>     ROUND(AVG(EXTRACT(EPOCH FROM
>         (review_answer_timestamp - review_creation_date)) / 3600)::numeric, 1) AS media_horas,
>     ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
>         ORDER BY EXTRACT(EPOCH FROM
>             (review_answer_timestamp - review_creation_date)) / 3600)
>         ::numeric, 1)                                                           AS mediana_horas,
>     COUNT(*)                                                                    AS qtd
> FROM olist_order_reviews_dataset
> WHERE review_answer_timestamp >= review_creation_date
> GROUP BY review_score
> ORDER BY review_score;
> ```

</details>

**Insight:** O tempo médio de resposta às avaliações é de **75,6 horas (3,1 dias)**, com mediana de **40,2h**. **71,9%** das respostas chegam em até 3 dias. Curiosamente, **avaliações negativas (nota 1) recebem resposta em 73,2h** — praticamente igual à média geral — sugerindo que não há priorização de reclamações urgentes. Avaliações nota 5 demoram levemente mais (**77h**), pois são a maioria e podem ser processadas em lote. Uma fila prioritária para notas 1–2 poderia reduzir o impacto reputacional de avaliações negativas.

---

## Análise Financeira

### Pergunta 22: Em quais categorias o frete representa a maior fatia do valor total pago?

**Objetivo:** Identificar onde o custo logístico penaliza mais o consumidor e cria fricção na conversão.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items    = pd.read_csv("olist_order_items_dataset.csv")
> products = pd.read_csv("olist_products_dataset.csv")
> transl   = pd.read_csv("product_category_name_translation.csv")
>
> df = (
>     items
>     .merge(products[['product_id', 'product_category_name']], on='product_id', how='left')
>     .merge(transl, on='product_category_name', how='left')
>     .assign(
>         categoria=lambda d: d['product_category_name_english'].fillna(d['product_category_name']),
>         pct_frete=lambda d: d['freight_value'] / (d['price'] + d['freight_value']) * 100
>     )
> )
>
> resultado = (
>     df.groupby('categoria').agg(
>         preco_medio=('price', 'mean'),
>         frete_medio=('freight_value', 'mean'),
>         pct_frete_media=('pct_frete', 'mean'),
>         qtd=('order_item_id', 'count')
>     )
>     .round(2)
> )
>
> filtrado = resultado[resultado['qtd'] >= 100]
> print("Categorias onde o frete pesa mais (%):")
> print(filtrado.sort_values('pct_frete_media', ascending=False).head(10))
>
> total_frete = items['freight_value'].sum()
> total_produto = items['price'].sum()
> print(f"\nFrete total: R$ {total_frete:,.2f} ({total_frete/(total_frete+total_produto)*100:.1f}% do total)")
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     COALESCE(t.product_category_name_english, p.product_category_name) AS categoria,
>     ROUND(AVG(i.price)::numeric, 2)          AS preco_medio,
>     ROUND(AVG(i.freight_value)::numeric, 2)  AS frete_medio,
>     ROUND(AVG(i.freight_value / NULLIF(i.price + i.freight_value, 0) * 100)
>           ::numeric, 1)                       AS pct_frete_media,
>     COUNT(*)                                  AS qtd
> FROM olist_order_items_dataset i
> LEFT JOIN olist_products_dataset p USING (product_id)
> LEFT JOIN product_category_name_translation t ON p.product_category_name = t.product_category_name
> GROUP BY categoria
> HAVING COUNT(*) >= 100
> ORDER BY pct_frete_media DESC
> LIMIT 10;
> ```

</details>

**Insight:** O frete representa **14,2% do total** movimentado na plataforma (R$ 2.251.909 sobre R$ 15.843.553). As categorias mais penalizadas são **eletrônicos (35,8%)** e **suprimentos natalinos (33,8%)** — itens baratos com frete alto. No extremo oposto, **computadores** têm frete de apenas **5,2%** do valor (produto caro + frete fixo). Nas categorias com frete >25%, ofertas de frete grátis ou subsidiado acima de determinado valor de compra poderiam aumentar significativamente a taxa de conversão.

---

### Pergunta 23: Como se distribui o parcelamento no cartão de crédito?

**Objetivo:** Entender o comportamento de financiamento dos clientes e a relação entre parcelamento e ticket médio.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> payments = pd.read_csv("olist_order_payments_dataset.csv")
>
> cc = payments[payments['payment_type'] == 'credit_card'].copy()
>
> faixas = pd.cut(cc['payment_installments'],
>                 bins=[0, 1, 3, 6, 12, 24],
>                 labels=['1x', '2-3x', '4-6x', '7-12x', '13-24x'])
>
> resultado = (
>     cc.groupby(faixas, observed=True)['payment_value']
>     .agg(qtd='count', valor_medio='mean')
>     .round(2)
> )
> resultado['pct'] = (resultado['qtd'] / resultado['qtd'].sum() * 100).round(1)
>
> print(f"Total transações cartão: {len(cc):,}")
> print(f"Média de parcelas: {cc['payment_installments'].mean():.2f}")
> print(resultado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     CASE
>         WHEN payment_installments = 1  THEN '1x'
>         WHEN payment_installments <= 3 THEN '2-3x'
>         WHEN payment_installments <= 6 THEN '4-6x'
>         WHEN payment_installments <= 12 THEN '7-12x'
>         ELSE '13-24x'
>     END AS faixa_parcelas,
>     COUNT(*)                                          AS qtd,
>     ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()
>           ::numeric, 1)                               AS pct,
>     ROUND(AVG(payment_value)::numeric, 2)             AS valor_medio
> FROM olist_order_payments_dataset
> WHERE payment_type = 'credit_card'
> GROUP BY faixa_parcelas
> ORDER BY valor_medio;
> ```

</details>

**Insight:** **33,1% das transações** no cartão são à vista (1x), com ticket médio de R$ 95,87. A parcela mais comum depois é **2–3x (29,7%)** com R$ 134. Compras **7–12x** têm ticket médio de **R$ 333** — 3,5x mais que à vista. Há um pico anômalo em **10 parcelas (6,94%)**, sugerindo que algum produto ou promoção específica empurrou para esse parcelamento exato. Campanhas de "parcele em 10x sem juros" claramente elevam o ticket médio e poderiam ser replicadas em outras categorias.

---

### Pergunta 24: Quantos pedidos combinam múltiplos métodos de pagamento?

**Objetivo:** Entender o uso de pagamentos combinados (ex.: cartão + voucher) e identificar padrões de uso de cupons.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> payments = pd.read_csv("olist_order_payments_dataset.csv")
>
> multi = payments.groupby('order_id').agg(
>     n_metodos=('payment_type', 'nunique'),
>     n_sequencias=('payment_sequential', 'max'),
>     valor_total=('payment_value', 'sum')
> ).reset_index()
>
> print(f"Pedidos com 1 método:  {(multi['n_metodos']==1).sum():,}")
> print(f"Pedidos com 2+ métodos: {(multi['n_metodos']>1).sum():,} "
>       f"({(multi['n_metodos']>1).sum()/len(multi)*100:.2f}%)")
>
> combos = (
>     payments.groupby('order_id')['payment_type']
>     .apply(lambda x: '+'.join(sorted(x.unique())))
>     .value_counts()
>     .head(10)
> )
> print("\nTop combinações de pagamento:")
> print(combos)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH metodos_por_pedido AS (
>     SELECT order_id,
>            COUNT(DISTINCT payment_type) AS n_metodos,
>            MAX(payment_sequential)       AS n_sequencias
>     FROM olist_order_payments_dataset
>     GROUP BY order_id
> )
> SELECT
>     CASE WHEN n_metodos = 1 THEN '1 método' ELSE '2+ métodos' END AS tipo,
>     COUNT(*)                                                        AS qtd,
>     ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()::numeric, 2)    AS pct
> FROM metodos_por_pedido
> GROUP BY tipo
> ORDER BY qtd DESC;
> ```

</details>

**Insight:** **2.246 pedidos (2,26%)** combinam mais de um método de pagamento. A combinação mais comum é **cartão de crédito + voucher (2.245 pedidos)** — praticamente todos os casos de multipagamento envolvem um cupom sendo aplicado parcialmente. O valor médio desses pedidos tende a ser mais alto (cupons usados em compras maiores). Um único pedido chegou a usar **11 sequências de pagamento**. Vouchers são usados quase exclusivamente como complemento, não como forma de pagamento principal.

---

## Análise de Cohort

### Pergunta 25: Qual é a taxa de retenção dos clientes por cohort de primeira compra?

**Objetivo:** Medir se clientes que chegaram em diferentes períodos têm comportamentos de recompra distintos.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv", parse_dates=['order_purchase_timestamp'])
> customers = pd.read_csv("olist_customers_dataset.csv")
>
> cust_orders = (
>     orders.merge(customers[['customer_id', 'customer_unique_id']], on='customer_id', how='left')
>     .assign(ano_mes=lambda df: df['order_purchase_timestamp'].dt.to_period('M'))
> )
>
> primeira_compra = (cust_orders.groupby('customer_unique_id')['ano_mes']
>                   .min().reset_index().rename(columns={'ano_mes': 'cohort'}))
>
> cohort_df = cust_orders.merge(primeira_compra, on='customer_unique_id', how='left')
> cohort_df['periodo'] = (cohort_df['ano_mes'] - cohort_df['cohort']).apply(lambda x: x.n)
>
> cohort_tab = (cohort_df.groupby(['cohort', 'periodo'])['customer_unique_id']
>               .nunique().unstack(fill_value=0))
> cohort_pct = cohort_tab.divide(cohort_tab[0], axis=0) * 100
>
> print("Retenção % — cohorts 2017 (primeiros 6 meses):")
> print(cohort_pct.loc['2017-01':'2017-12', 0:5].round(1))
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH primeira_compra AS (
>     SELECT
>         c.customer_unique_id,
>         DATE_TRUNC('month', MIN(o.order_purchase_timestamp)) AS cohort
>     FROM olist_orders_dataset o
>     JOIN olist_customers_dataset c USING (customer_id)
>     GROUP BY c.customer_unique_id
> ),
> cohort_data AS (
>     SELECT
>         pc.cohort,
>         DATE_PART('month', AGE(DATE_TRUNC('month', o.order_purchase_timestamp), pc.cohort)) AS periodo,
>         c.customer_unique_id
>     FROM olist_orders_dataset o
>     JOIN olist_customers_dataset c USING (customer_id)
>     JOIN primeira_compra pc USING (customer_unique_id)
> )
> SELECT
>     TO_CHAR(cohort, 'YYYY-MM')  AS cohort,
>     periodo,
>     COUNT(DISTINCT customer_unique_id) AS clientes
> FROM cohort_data
> GROUP BY cohort, periodo
> ORDER BY cohort, periodo;
> ```

</details>

**Insight:** A análise de cohort confirma de forma contundente a baixa retenção: a **retenção média no mês 1** (clientes que voltaram a comprar no mês seguinte à primeira compra) é de apenas **4,4%** — e esse número cai progressivamente nos meses seguintes (geralmente abaixo de 0,5% no mês 5). Não há diferença significativa entre cohorts de diferentes períodos: clientes de jan/2017 e ago/2017 têm o mesmo comportamento. Isso indica que o problema é estrutural — não há mecanismo de retenção ativo — e não depende de sazonalidade ou maturidade da plataforma.

---

## Análise de Cancelamentos

### Pergunta 26: Em quais estados e categorias os cancelamentos são mais frequentes?

**Objetivo:** Identificar padrões geográficos e de produto nos cancelamentos para reduzir essa taxa.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> orders    = pd.read_csv("olist_orders_dataset.csv")
> customers = pd.read_csv("olist_customers_dataset.csv")
> items     = pd.read_csv("olist_order_items_dataset.csv")
> products  = pd.read_csv("olist_products_dataset.csv")
> transl    = pd.read_csv("product_category_name_translation.csv")
>
> canceled = orders[orders['order_status'] == 'canceled']
> total_por_estado = (
>     orders.merge(customers[['customer_id', 'customer_state']], on='customer_id', how='left')
>     .groupby('customer_state')['order_id'].count()
> )
>
> taxa_estado = (
>     canceled.merge(customers[['customer_id', 'customer_state']], on='customer_id', how='left')
>     .groupby('customer_state')['order_id'].count()
>     .rename('qtd_cancel')
>     .to_frame()
>     .assign(total=total_por_estado,
>             taxa_pct=lambda df: (df['qtd_cancel'] / df['total'] * 100).round(2))
>     .sort_values('taxa_pct', ascending=False)
>     .head(10)
> )
> print(f"Total cancelamentos: {len(canceled)}")
> print(taxa_estado)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> WITH totais AS (
>     SELECT c.customer_state, COUNT(*) AS total
>     FROM olist_orders_dataset o
>     JOIN olist_customers_dataset c USING (customer_id)
>     GROUP BY c.customer_state
> ),
> cancelados AS (
>     SELECT c.customer_state, COUNT(*) AS qtd_cancel
>     FROM olist_orders_dataset o
>     JOIN olist_customers_dataset c USING (customer_id)
>     WHERE o.order_status = 'canceled'
>     GROUP BY c.customer_state
> )
> SELECT
>     t.customer_state,
>     COALESCE(c.qtd_cancel, 0)                          AS qtd_cancel,
>     t.total,
>     ROUND(COALESCE(c.qtd_cancel, 0) * 100.0 / t.total
>           ::numeric, 2)                                 AS taxa_pct
> FROM totais t
> LEFT JOIN cancelados c USING (customer_state)
> ORDER BY taxa_pct DESC;
> ```

</details>

**Insight:** Há **625 cancelamentos** no total (0,63% de todos os pedidos). **Roraima (2,17%)** e **Rondônia (1,19%)** têm as maiores taxas, mas com volumes pequenos. **SP** concentra o maior número absoluto (327 cancelamentos) por ser o maior estado. Por categoria, **sports_leisure (47)**, **housewares (37)** e **health_beauty (36)** têm o maior volume de cancelamentos — as mesmas categorias top em receita, o que é esperado pelo volume. A taxa de cancelamento por categoria é baixa o suficiente para não ser um alerta crítico, mas merece monitoramento contínuo.

---

## Análise de Produtos

### Pergunta 27: A quantidade de fotos e o tamanho da descrição impactam a satisfação do cliente?

**Objetivo:** Avaliar se investir em conteúdo de produto (mais fotos, descrições mais completas) melhora as avaliações.

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white"><img src="https://img.shields.io/badge/Pandas-Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python"></picture> &nbsp;<b>Pandas</b></summary>

<br>

> ```python
> import pandas as pd
>
> items    = pd.read_csv("olist_order_items_dataset.csv")
> products = pd.read_csv("olist_products_dataset.csv")
> reviews  = pd.read_csv("olist_order_reviews_dataset.csv")
>
> df = (
>     items[['order_id', 'product_id']]
>     .merge(products[['product_id', 'product_photos_qty', 'product_description_lenght']],
>            on='product_id', how='left')
>     .merge(reviews[['order_id', 'review_score']], on='order_id', how='left')
>     .dropna(subset=['product_photos_qty', 'review_score'])
> )
>
> score_por_foto = (
>     df.groupby('product_photos_qty')['review_score']
>     .agg(media_score='mean', qtd='count')
>     .round(3)
> )
>
> faixa_desc = pd.cut(df['product_description_lenght'],
>                     bins=[0, 200, 500, 1000, 2000, 4000],
>                     labels=['<200', '200-500', '500-1000', '1000-2000', '>2000'])
> score_por_desc = df.groupby(faixa_desc, observed=True)['review_score'].agg(['mean', 'count']).round(3)
>
> corr_fotos = df['product_photos_qty'].corr(df['review_score'])
> corr_desc  = df['product_description_lenght'].corr(df['review_score'])
>
> print(score_por_foto[score_por_foto['qtd'] >= 50])
> print(f"\nCorrelação fotos × score: {corr_fotos:.3f}")
> print(f"Correlação descrição × score: {corr_desc:.3f}")
> print("\nScore por faixa de descrição:")
> print(score_por_desc)
> ```

</details>

<details>
<summary><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white"><img src="https://img.shields.io/badge/PostgreSQL-SQL-4169E1?style=flat&logo=postgresql&logoColor=white" alt="SQL"></picture> &nbsp;<b>PostgreSQL</b></summary>

<br>

> ```sql
> SELECT
>     p.product_photos_qty,
>     ROUND(AVG(r.review_score)::numeric, 3) AS media_score,
>     COUNT(*)                                AS qtd,
>     ROUND(CORR(p.product_photos_qty::float, r.review_score::float)
>           ::numeric, 3)                     AS correlacao
> FROM olist_order_items_dataset i
> LEFT JOIN olist_products_dataset p USING (product_id)
> LEFT JOIN olist_order_reviews_dataset r USING (order_id)
> WHERE p.product_photos_qty IS NOT NULL
> GROUP BY p.product_photos_qty
> HAVING COUNT(*) >= 50
> ORDER BY p.product_photos_qty;
> ```

</details>

**Insight:** A correlação entre número de fotos e review score é de apenas **0,023** — praticamente nula. O mesmo vale para o tamanho da descrição (**0,013**). Produtos com **1 foto** têm score médio de **3,999**, e produtos com **5 fotos** chegam a **4,156** — uma diferença de apenas 0,157 pontos. Isso sugere que **o conteúdo visual do produto não é o driver da satisfação** — a experiência de entrega (prazo, estado do produto) domina. No entanto, mais fotos correlacionam levemente com categorias de maior qualidade, o que pode explicar o efeito marginal observado.

---

## Conclusão

| Tema | Perguntas | Principal Descoberta |
|------|-----------|----------------------|
| Receita/Valores | 1, 2, 3 | Faturamento de R$ 16M; health_beauty lidera; cartão responde por 73,9% das transações |
| Localização | 4, 5, 6 | SP concentra 42% dos pedidos e 64% dos vendedores — risco de hiperdependência |
| Temporal | 7, 8, 9 | Crescimento 23x em 2 anos; pico na Black Friday; segunda-feira às 16h tem mais compras |
| Comportamento | 10, 11, 12 | 97% entregues; apenas 3,12% recompram; entrega chega 11 dias antes do estimado |
| Correlações | 13, 14 | Prazo impacta nota (-0,334); peso é o principal driver do frete (0,610) |
| Segmentação RFM | 15 | 41% são Potential Loyalists; Champions têm ticket de R$ 334 e compra recente |
| Vendedores | 16, 17 | 17,5% dos vendedores geram 80% da receita; pior vendedor tem score 2,20 com 136 avaliações |
| Logística | 18, 19 | 8,11% dos pedidos chegam atrasados; tempo médio de preparo é 2,9 dias (Maranhão: 4,7 dias) |
| Reviews | 20, 21 | office_furniture tem score 3,62 e 22,6% de negativas; respostas levam 3,1 dias em média |
| Financeiro | 22, 23, 24 | Frete = 14,2% do total; compras em 10x têm ticket 3,5x maior que à vista; 2,26% usam múltiplos métodos |
| Cohort | 25 | Retenção no mês 1 é de apenas 4,4% — consistente em todos os cohorts, problema estrutural |
| Cancelamentos | 26 | 625 cancelamentos (0,63%); RR tem maior taxa (2,17%); sports_leisure lidera em volume |
| Produtos | 27 | Fotos e tamanho da descrição têm correlação quase nula com o score (0,023 e 0,013) |

> **Insight geral:** A Olist opera com excelência logística (97% de entrega, 11 dias de antecedência) e crescimento acelerado (23x em 2 anos). Os dois maiores desafios estruturais são a **hiperdependência geográfica de SP** (64% da receita de vendedores) e a **baixíssima retenção** (retenção mês 1 de apenas 4,4% em todos os cohorts). A análise de Pareto mostra que 543 vendedores (17,5%) geram 80% da receita — risco de concentração. A satisfação do cliente é dominada pelo prazo de entrega (correlação -0,334 com o score), não pelo conteúdo visual do produto. Converter os 38.399 Potential Loyalists e reduzir o tempo de preparo dos vendedores mais lentos são as alavancas de maior impacto imediato.

> **Insight geral:** A Olist opera com excelência logística (97% de entrega, 11 dias de antecedência) e crescimento acelerado (23x em 2 anos). Os dois maiores desafios estruturais são a **hiperdependência geográfica de SP** (oferta e demanda) e a **baixíssima retenção** (96,88% compram apenas uma vez). Converter os 38.399 Potential Loyalists com campanhas personalizadas na janela segunda-feira/16h pode aumentar significativamente o LTV da base sem necessidade de aquisição de novos clientes.

---

## Estrutura do Projeto

```
brazilian_e-commerce/
├── olist_customers_dataset.csv
├── olist_geolocation_dataset.csv
├── olist_order_items_dataset.csv
├── olist_order_payments_dataset.csv
├── olist_order_reviews_dataset.csv
├── olist_orders_dataset.csv
├── olist_products_dataset.csv
├── olist_sellers_dataset.csv
├── product_category_name_translation.csv
├── dataset_schema.md
├── README.md
└── analise_brazilian_ecommerce.ipynb
```

## Tecnologias Utilizadas

- **Python 3.10+**
- **Pandas** — manipulação e análise de dados
- **Jupyter Notebook** — ambiente interativo
- **PostgreSQL** — queries relacionais documentadas
- **Kaggle API** — download automatizado do dataset

## Como Executar

```bash
# 1. Clone o repositório
git clone <repo-url>
cd brazilian_e-commerce

# 2. Instale as dependências
pip install pandas jupyter python-dotenv kaggle

# 3. (Opcional) Configure a Kaggle API
echo '{"username":"SEU_USER","key":"SUA_KEY"}' > ~/.kaggle/kaggle.json

# 4. Abra o notebook
jupyter notebook analise_brazilian_ecommerce.ipynb
```
