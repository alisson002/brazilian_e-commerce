# Brazilian E-Commerce — Análise Exploratória de Dados (Olist)

Dataset público da Olist com **99.441 pedidos** realizados entre 2016 e 2018 no e-commerce brasileiro. Este projeto responde 15 perguntas analíticas com Pandas e PostgreSQL, cobrindo receita, geografia, comportamento do consumidor, tendências temporais e segmentação de clientes.

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

## Conclusão

| Tema | Perguntas | Principal Descoberta |
|------|-----------|----------------------|
| Receita/Valores | 1, 2, 3 | Faturamento de R$ 16M; health_beauty lidera; cartão responde por 73,9% das transações |
| Localização | 4, 5, 6 | SP concentra 42% dos pedidos e 64% dos vendedores — risco de hiperdependência |
| Temporal | 7, 8, 9 | Crescimento 23x em 2 anos; pico na Black Friday; segunda-feira às 16h tem mais compras |
| Comportamento | 10, 11, 12 | 97% entregues; apenas 3,12% recompram; entrega chega 11 dias antes do estimado |
| Correlações | 13, 14 | Prazo impacta nota (-0,334); peso é o principal driver do frete (0,610) |
| Segmentação RFM | 15 | 41% são Potential Loyalists; Champions têm ticket de R$ 334 e compra recente |

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
