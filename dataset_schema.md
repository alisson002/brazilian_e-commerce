# Brazilian E-Commerce Dataset Schema — Olist

## Datasets disponíveis

| Dataset | Cor no diagrama | Descrição |
|---|---|---|
| `olist_orders_dataset` | Vermelho | Tabela central de pedidos |
| `olist_order_items_dataset` | Laranja | Itens de cada pedido |
| `olist_order_payments_dataset` | Cinza | Pagamentos dos pedidos |
| `olist_order_reviews_dataset` | Roxo | Avaliações dos pedidos |
| `olist_products_dataset` | Amarelo | Catálogo de produtos |
| `olist_sellers_dataset` | Verde (direita) | Dados dos vendedores |
| `olist_order_customer_dataset` | Verde (baixo) | Dados dos clientes |
| `olist_geolocation_dataset` | Azul | Dados de geolocalização por CEP |

---

## Conexões entre datasets

### olist_orders_dataset (tabela central)

| Conecta com | Coluna de junção |
|---|---|
| `olist_order_payments_dataset` | `order_id` |
| `olist_order_reviews_dataset` | `order_id` |
| `olist_order_items_dataset` | `order_id` |
| `olist_order_customer_dataset` | `customer_id` |

### olist_order_items_dataset

| Conecta com | Coluna de junção |
|---|---|
| `olist_orders_dataset` | `order_id` |
| `olist_products_dataset` | `product_id` |
| `olist_sellers_dataset` | `seller_id` |

### olist_order_payments_dataset

| Conecta com                  | Coluna de junção |
| ---------------------------- | ---------------- |
| `olist_orders_dataset`       | `order_id`       |

### olist_order_reviews_dataset

| Conecta com                  | Coluna de junção |
| ---------------------------- | ---------------- |
| `olist_orders_dataset`       | `order_id`       |

### olist_products_dataset

| Conecta com                  | Coluna de junção |
| ---------------------------- | ---------------- |
| `olist_order_items_dataset`  | `product_id`     |

### olist_sellers_dataset

| Conecta com | Coluna de junção |
|---|---|
| `olist_order_items_dataset` | `seller_id` |
| `olist_geolocation_dataset` | `zip_code_prefix` |

### olist_order_customer_dataset

| Conecta com | Coluna de junção |
|---|---|
| `olist_orders_dataset` | `customer_id` |
| `olist_geolocation_dataset` | `zip_code_prefix` |

### olist_geolocation_dataset

| Conecta com | Coluna de junção |
|---|---|
| `olist_sellers_dataset` | `zip_code_prefix` |
| `olist_order_customer_dataset` | `zip_code_prefix` |

---

## Resumo de todas as relações

```
olist_orders_dataset
  ├── order_id      → olist_order_payments_dataset
  ├── order_id      ↔ olist_order_reviews_dataset
  ├── order_id      ↔ olist_order_items_dataset
  └── customer_id   → olist_order_customer_dataset

olist_order_items_dataset
  ├── product_id    → olist_products_dataset
  └── seller_id     ↔ olist_sellers_dataset

olist_sellers_dataset
  └── zip_code_prefix ↔ olist_geolocation_dataset

olist_order_customer_dataset
  └── zip_code_prefix ↔ olist_geolocation_dataset
```

---

## Chaves de junção por coluna

| Coluna | Tabelas envolvidas |
|---|---|
| `order_id` | orders ↔ order_items, order_payments, order_reviews |
| `customer_id` | orders ↔ order_customer |
| `product_id` | order_items ↔ products |
| `seller_id` | order_items ↔ sellers |
| `zip_code_prefix` | sellers ↔ geolocation, order_customer ↔ geolocation |
