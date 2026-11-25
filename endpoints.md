# Endpoints da API — Backend da pizzaria

## Informações gerais

- **Prefixo:** `/api/v1`
- **Formato:** JSON, exceto o cadastro de produto, que usa `multipart/form-data`.
- **Autenticação:** endpoints protegidos exigem `Authorization: Bearer <token>`.
- **Validação:** as entradas são validadas com Zod. Falhas retornam `400 Bad Request` no formato:

```json
{
  "message": "Erro de validação",
  "details": [
    {
      "path": "campo",
      "message": "Descrição do erro"
    }
  ]
}
```

- Algumas regras chamam `trim()` durante a validação, mas o middleware não substitui `req.body`/`req.query` pelos valores parseados. Portanto, o trim só é persistido quando o controller ou serviço também o aplica explicitamente.
- Erros `AppError` retornam `{ "error": "mensagem" }` com o status definido pelo serviço. Erros não tratados retornam `500` com `{ "error": "Internal Server Error" }`.
- Usuários comuns têm papel `STAFF`; operações administrativas exigem o papel `ADMIN`.

## Usuários e sessão

### `POST /api/v1/users`

Cria um usuário. Não exige autenticação.

**Body — JSON**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `name` | string | Sim | Após trim, mínimo de 3 caracteres |
| `email` | string | Sim | Após trim, deve ser um e-mail válido |
| `password` | string | Sim | Mínimo de 6 caracteres |

```json
{
  "name": "Nome do usuário",
  "email": "nome@exemplo.com",
  "password": "senha123"
}
```

**Sucesso — `201 Created`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `email` | string |
| `role` | string (`STAFF` para novos usuários) |

Não retorna a senha.

**Erros:** `400` entrada inválida; e-mail duplicado atualmente resulta em `500` (`Email já cadastrado` é lançado como erro genérico).

### `POST /api/v1/session`

Autentica um usuário e retorna um token JWT. Não exige autenticação.

**Body — JSON**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `email` | string | Sim | Após trim, deve ser um e-mail válido |
| `password` | string | Sim | Mínimo de 6 caracteres |

```json
{
  "email": "nome@exemplo.com",
  "password": "senha123"
}
```

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `email` | string |
| `role` | string (`ADMIN` ou `STAFF`) |
| `token` | string (JWT, validade de 2 dias) |

**Erros:** `400` entrada inválida; `401` usuário ou senha inválidos.

### `GET /api/v1/me`

Retorna o usuário associado ao token.

**Autenticação:** obrigatória.

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `email` | string |
| `role` | string (`ADMIN` ou `STAFF`) |

**Erros:** `401` token ausente ou inválido; usuário não encontrado atualmente resulta em `500` por ser lançado como erro genérico.

## Categorias

### `POST /api/v1/categories`

Cria uma categoria.

**Autenticação:** obrigatória; somente `ADMIN`.

**Body — JSON**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `name` | string | Sim | Após trim, mínimo de 3 caracteres |
| `description` | string | Sim | Após trim, mínimo de 3 caracteres |

```json
{
  "name": "Pizzas",
  "description": "Pizzas tradicionais"
}
```

**Sucesso — `201 Created`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `description` | string |

**Erros:** `400` entrada inválida; `401` token ausente/inválido; `403` usuário não é administrador; `409` categoria com o mesmo nome já cadastrada.

### `GET /api/v1/categories`

Lista categorias em ordem crescente pelo nome. Retorna `[]` se não houver categorias.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Sucesso — `200 OK`** — array de:

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `description` | string |

**Erros:** `401` token ausente ou inválido.

## Produtos

### `POST /api/v1/products`

Cria um produto e envia sua imagem ao Cloudinary.

**Autenticação:** obrigatória; somente `ADMIN`.

**Content-Type:** `multipart/form-data`

| Campo | Tipo | Obrigatório | Regras |
|---|---|---:|---|
| `name` | string | Sim | Após trim, mínimo de 3 caracteres |
| `description` | string | Sim | Após trim, mínimo de 3 caracteres |
| `price` | número ou texto numérico | Sim | Convertido para número inteiro finito maior que zero |
| `category_id` | string | Sim | Após trim, não pode ser vazio; deve identificar uma categoria existente |
| `file` | arquivo | Sim | Imagem JPEG, JPG, PNG ou WebP; máximo de 4 MiB |

O nome é persistido após trim. A API guarda a URL segura do Cloudinary no campo `banner`.

**Sucesso — `201 Created`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `description` | string |
| `price` | number (inteiro) |
| `banner` | string (URL da imagem) |
| `category_id` | string (UUID da categoria) |
| `disabled` | boolean (`false` na criação) |

**Erros:** `400` dados inválidos, arquivo ausente ou tipo de arquivo inválido; `401` token ausente/inválido; `403` não administrador; `404` categoria inexistente; `409` produto com nome normalizado duplicado; `413` arquivo acima de 4 MiB.

### `GET /api/v1/products`

Lista produtos por estado, ordenados pelo nome.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `disabled` | string | Não | Aceita somente `true` ou `false`; padrão `false` |

Exemplos: `/api/v1/products` (ativos), `/api/v1/products?disabled=false` e `/api/v1/products?disabled=true`.

**Sucesso — `200 OK`** — array de produtos com:

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `name` | string |
| `description` | string |
| `price` | number (inteiro) |
| `banner` | string (URL da imagem) |
| `category_id` | string (UUID) |
| `disabled` | boolean |

**Erros:** `400` valor inválido para `disabled`; `401` token ausente ou inválido.

### `GET /api/v1/category/product`

Lista produtos associados a uma categoria, incluindo ativos e desativados, em ordem crescente pelo nome.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `category_id` | string | Sim | UUID válido de uma categoria |

**Sucesso — `200 OK`** — array de produtos com `id`, `name`, `description`, `price`, `banner`, `category_id` e `disabled`, conforme `GET /api/v1/products`.

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` categoria inexistente.

### `DELETE /api/v1/products/:id`

Desativa o produto (`disabled: true`); não remove o registro nem a imagem do Cloudinary. Repetir a operação mantém o produto desativado.

**Autenticação:** obrigatória; somente `ADMIN`.

**Path parameter**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `id` | string | Sim | UUID válido do produto |

**Sucesso — `200 OK`**

```json
{
  "message": "Produto deletado com sucesso"
}
```

**Erros:** `400` UUID inválido; `401` token ausente/inválido; `403` não administrador; `404` produto inexistente.

## Pedidos

### `POST /api/v1/orders`

Cria um pedido em rascunho.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Body — JSON**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `table` | number | Sim | Inteiro maior que zero |
| `name` | string | Sim | Após trim, não pode ser vazio |

```json
{
  "table": 3,
  "name": "Cliente"
}
```

**Sucesso — `201 Created`**

| Propriedade | Tipo | Valor inicial |
|---|---|---|
| `id` | string (UUID) | Gerado |
| `table` | number (inteiro) | Informado |
| `name` | string | Informado após trim |
| `status` | boolean | `false` |
| `draft` | boolean | `true` |
| `items` | array | `[]` |

**Erros:** `400` entrada inválida; `401` token ausente ou inválido.

### `GET /api/v1/orders`

Lista pedidos em ordem crescente pelo nome do cliente, incluindo itens e informações básicas dos produtos.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `draft` | string | Sim | Aceita somente `true` ou `false` |

**Sucesso — `200 OK`** — array de pedidos:

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `table` | number (inteiro) |
| `name` | string |
| `status` | boolean |
| `draft` | boolean |
| `items` | array de itens descritos abaixo |

Cada item contém:

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `amount` | number (inteiro) |
| `order_id` | string (UUID) |
| `product_id` | string (UUID) |
| `product` | objeto de produto descrito abaixo |

O objeto `product` contém `id` (string/UUID), `name` (string), `price` (number), `description` (string) e `banner` (string/URL).

**Erros:** `400` parâmetro `draft` ausente ou inválido; `401` token ausente ou inválido.

### `POST /api/v1/order/add`

Adiciona um produto ao pedido em rascunho. Se já houver um item do mesmo produto, incrementa sua quantidade.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Body — JSON**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `order_id` | string | Sim | UUID válido |
| `product_id` | string | Sim | UUID válido |
| `amount` | number | Sim | Inteiro maior que zero |

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID do item) |
| `amount` | number (inteiro, quantidade total após a operação) |
| `order_id` | string (UUID) |
| `product_id` | string (UUID) |
| `product` | objeto com `id`, `name`, `price`, `description` e `banner` |

**Erros:** `400` entrada inválida; `401` token ausente/inválido; `404` pedido ou produto inexistente; `409` pedido não está em rascunho ou produto está desativado.

### `DELETE /api/v1/order/remove`

Remove um item de pedido pelo seu ID.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `item_id` | string | Sim | UUID válido do item |

**Sucesso — `200 OK`**

```json
{
  "message": "Item removido do pedido com sucesso"
}
```

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` item inexistente.

### `GET /api/v1/order/detail`

Retorna os detalhes de um pedido, incluindo data de criação e itens.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `order_id` | string | Sim | UUID válido do pedido |

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `table` | number (inteiro) |
| `name` | string |
| `status` | boolean |
| `draft` | boolean |
| `createdAt` | string (data/hora ISO 8601) |
| `items` | array de itens descritos abaixo |

Cada item contém `id` (string/UUID), `amount` (number inteiro) e `product`. O objeto `product` contém `id`, `name`, `price`, `description` e `banner`. Nesta resposta os itens não incluem `order_id` nem `product_id`.

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` pedido inexistente.

### `PUT /api/v1/order/send`

Marca o pedido como enviado para a cozinha, definindo `draft` como `false`. Não altera `status`.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `order_id` | string | Sim | UUID válido do pedido |

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `table` | number (inteiro) |
| `name` | string |
| `draft` | boolean (`false` após a operação) |
| `status` | boolean |
| `createdAt` | string (data/hora ISO 8601) |

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` pedido inexistente.

### `PUT /api/v1/order/finish`

Marca o pedido como finalizado, definindo `status` como `true`. Não altera `draft`.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `order_id` | string | Sim | UUID válido do pedido |

**Sucesso — `200 OK`**

| Propriedade | Tipo |
|---|---|
| `id` | string (UUID) |
| `table` | number (inteiro) |
| `name` | string |
| `draft` | boolean |
| `status` | boolean (`true` após a operação) |
| `createdAt` | string (data/hora ISO 8601) |

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` pedido inexistente.

### `DELETE /api/v1/order`

Exclui um pedido somente se estiver em rascunho e não possuir itens. Para excluir um pedido com itens, remova-os primeiro por `DELETE /api/v1/order/remove`.

**Autenticação:** obrigatória; qualquer usuário autenticado.

**Query parameters**

| Propriedade | Tipo | Obrigatória | Regras |
|---|---|---:|---|
| `order_id` | string | Sim | UUID válido do pedido |

**Sucesso — `200 OK`**

```json
{
  "message": "Pedido excluído com sucesso"
}
```

**Erros:** `400` UUID inválido ou parâmetro ausente; `401` token ausente ou inválido; `404` pedido inexistente; `409` pedido não está em rascunho ou ainda possui itens.
