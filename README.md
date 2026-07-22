# Documento Técnico — APIs de Integração
## App de Serviços Condominiais + Portal de Gerenciamento (Portaria)

**Versão:** 1.0
**Data:** 22/07/2026

---

## 1. Visão Geral

Este documento define as APIs REST necessárias para integrar o **aplicativo mobile** (uso dos moradores) com o **portal de gerenciamento** (uso da portaria/administração). Ambos os clientes consomem o mesmo backend, diferenciando-se pelo **papel do usuário** (`morador` ou `portaria`) retornado na autenticação, que determina quais endpoints e ações estão disponíveis.

### 1.1 Escopo funcional (baseado nos protótipos de tela)

| Módulo | Tela do App | Uso no Portal |
|---|---|---|
| Autenticação | Login / Cadastro | Login da portaria |
| Painel | Painel principal | Dashboard administrativo |
| Boletos | Lista de boletos | Cadastro/emissão de boletos |
| Reservas | Lista e reserva de áreas comuns | Aprovação/gestão de reservas |
| Autorização de visitantes | Cadastro de autorização | Consulta/validação na portaria |
| Mural | Lista de comunicados | Publicação de comunicados |
| Prestação de contas | Extrato de receitas/despesas | Lançamento de receitas/despesas |
| Mudanças | Cadastro de mudança | Aprovação/agenda de mudanças |
| Contatos | Lista de contatos úteis | Cadastro de contatos |

### 1.2 Padrões gerais

- **Protocolo:** HTTPS, REST, payloads em JSON.
- **Base URL:** `https://api.condservice.com/v1`
- **Autenticação:** Bearer Token (JWT) no header `Authorization: Bearer {token}`.
- **Formato de datas:** ISO 8601 (`yyyy-MM-dd` ou `yyyy-MM-dd'T'HH:mm:ssZ`).
- **Valores monetários:** número decimal em reais (ex.: `1500.00`), sem formatação de moeda.
- **Paginação:** `?page=1&size=20` nos endpoints de listagem, retornando `{ "content": [...], "page": 1, "totalPages": N, "totalElements": N }`.
- **Papéis (roles):** `MORADOR`, `PORTARIA`, `ADMIN`.

### 1.3 Padrão de resposta de erro

```json
{
  "timestamp": "2026-07-22T10:00:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Campos obrigatórios não preenchidos",
  "path": "/v1/autorizacoes"
}
```

Códigos utilizados: `200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `500 Internal Server Error`.

---

## 2. Autenticação

### 2.1 Login
`POST /auth/login`

**Request**
```json
{
  "email": "morador@example.com",
  "senha": "********"
}
```

**Response 200**
```json
{
  "accessToken": "jwt...",
  "refreshToken": "jwt...",
  "usuario": {
    "id": "u123",
    "nome": "Seu nome aqui",
    "role": "MORADOR",
    "apartamento": "101"
  }
}
```

### 2.2 Cadastro
`POST /auth/cadastro`

**Request**
```json
{
  "nome": "string",
  "email": "string",
  "senha": "string",
  "apartamento": "string",
  "bloco": "string"
}
```
**Response:** `201 Created` com o objeto do usuário criado (sem senha).

### 2.3 Refresh token
`POST /auth/refresh` — recebe `refreshToken`, retorna novo `accessToken`.

---

## 3. Painel (Home)

`GET /painel`

Retorna dados agregados para montar a tela inicial (nome do morador, atalhos e contadores — ex. novos comunicados, boletos em aberto).

**Response 200**
```json
{
  "usuario": { "nome": "string", "apartamento": "string" },
  "boletosEmAberto": 2,
  "comunicadosNaoLidos": 1,
  "atalhos": ["boletos", "reserva", "autorizacao", "mural", "prestacaoContas", "mudancas", "contatos"]
}
```

---

## 4. Boletos

| Ação | Endpoint | Uso |
|---|---|---|
| Listar boletos do morador | `GET /boletos` | App |
| Detalhar boleto | `GET /boletos/{id}` | App |
| Emitir boleto (individual ou em lote) | `POST /boletos` | Portal |
| Atualizar status (pago/cancelado) | `PATCH /boletos/{id}/status` | Portal |

**Modelo `Boleto`**
```json
{
  "id": "string",
  "descricao": "Taxa Condomínio Abril",
  "valor": 1500.00,
  "vencimento": "2026-05-18",
  "status": "PENDENTE",
  "apartamento": "101",
  "linkPagamento": "string",
  "codigoBarras": "string"
}
```

`status`: `PENDENTE`, `PAGO`, `VENCIDO`, `CANCELADO`.

`POST /boletos` (Portal) aceita lista de apartamentos-alvo:
```json
{
  "descricao": "string",
  "valor": 1500.00,
  "vencimento": "2026-06-10",
  "apartamentos": ["101", "102", "*"]
}
```
`"*"` gera o boleto para todos os apartamentos ativos.

---

## 5. Reservas de Áreas Comuns

| Ação | Endpoint | Uso |
|---|---|---|
| Listar áreas disponíveis | `GET /areas-comuns` | App |
| Listar reservas do morador | `GET /reservas` | App |
| Criar solicitação de reserva | `POST /reservas` | App |
| Cancelar reserva | `DELETE /reservas/{id}` | App |
| Listar todas as reservas (agenda) | `GET /reservas?apartamento=&area=&data=` | Portal |
| Aprovar/recusar reserva | `PATCH /reservas/{id}/status` | Portal |

**Modelo `AreaComum`**
```json
{ "id": "string", "nome": "Academia", "disponivel": true }
```
Áreas do protótipo: Academia, Brinquedoteca, Churrasqueira, Salão de Festa, Salão de Jogos.

**Modelo `Reserva`**
```json
{
  "id": "string",
  "areaId": "string",
  "areaNome": "Churrasqueira",
  "apartamento": "101",
  "data": "2026-08-15",
  "horaInicio": "12:00",
  "horaFim": "16:00",
  "status": "PENDENTE"
}
```
`status`: `PENDENTE`, `APROVADA`, `RECUSADA`, `CANCELADA`.

---

## 6. Autorização de Visitantes

| Ação | Endpoint | Uso |
|---|---|---|
| Cadastrar autorização | `POST /autorizacoes` | App |
| Listar autorizações do apartamento | `GET /autorizacoes` | App |
| Consultar autorizações do dia / validar entrada | `GET /autorizacoes?data=&status=` | Portal |
| Confirmar entrada do visitante | `PATCH /autorizacoes/{id}/checkin` | Portal |

**Request `POST /autorizacoes`**
```json
{
  "nomeCompleto": "string",
  "documento": "string",
  "apartamento": "101",
  "dataValidade": "2026-12-31"
}
```

**Validações e respostas:**
- `201 Created` + corpo do registro → mensagem de sucesso ("Autorização realizada") exibida pelo app.
- `400 Bad Request` quando `nomeCompleto`, `documento` ou `apartamento` estiverem vazios → mensagem de erro ("Todos os dados são obrigatórios") exibida pelo app.

**Modelo `Autorizacao`**
```json
{
  "id": "string",
  "nomeCompleto": "string",
  "documento": "string",
  "apartamento": "101",
  "dataValidade": "2026-12-31",
  "status": "PENDENTE",
  "checkinEm": null
}
```
`status`: `PENDENTE`, `UTILIZADA`, `EXPIRADA`.

---

## 7. Mural de Comunicados

| Ação | Endpoint | Uso |
|---|---|---|
| Listar comunicados | `GET /comunicados` | App |
| Publicar comunicado | `POST /comunicados` | Portal |
| Editar/remover comunicado | `PUT /comunicados/{id}` / `DELETE /comunicados/{id}` | Portal |

**Modelo `Comunicado`**
```json
{
  "id": "string",
  "titulo": "Novo Bicicletário",
  "corpo": "string",
  "publicadoEm": "2026-07-01T09:00:00Z",
  "fixado": false
}
```

---

## 8. Prestação de Contas

| Ação | Endpoint | Uso |
|---|---|---|
| Obter resumo (totais) + extrato | `GET /prestacao-contas?mes=&ano=` | App |
| Registrar lançamento (receita/despesa) | `POST /lancamentos` | Portal |
| Editar/remover lançamento | `PUT /lancamentos/{id}` / `DELETE /lancamentos/{id}` | Portal |

**Response `GET /prestacao-contas`**
```json
{
  "totalRecebido": 5000.00,
  "totalGasto": 3200.00,
  "saldoAtual": 1800.00,
  "lancamentos": [
    { "id": "1", "descricao": "Compra de material", "tipo": "DESPESA", "valor": 500.00, "data": "2026-05-02" },
    { "id": "2", "descricao": "Mensalidade condomínio", "tipo": "RECEITA", "valor": 1500.00, "data": "2026-05-05" },
    { "id": "3", "descricao": "Manutenção elevador", "tipo": "DESPESA", "valor": 1200.00, "data": "2026-05-10" }
  ]
}
```
`tipo`: `RECEITA`, `DESPESA`.

---

## 9. Mudanças

| Ação | Endpoint | Uso |
|---|---|---|
| Cadastrar solicitação de mudança | `POST /mudancas` | App |
| Listar mudanças do morador | `GET /mudancas` | App |
| Listar/agendar todas as mudanças | `GET /mudancas?data=` | Portal |
| Aprovar mudança | `PATCH /mudancas/{id}/status` | Portal |

**Request `POST /mudancas`**
```json
{
  "nomeResponsavel": "string",
  "apartamento": "string",
  "dataMudanca": "2026-08-01",
  "tipo": "ENTRADA"
}
```
`tipo`: `ENTRADA`, `SAIDA`. Resposta `201 Created` dispara a mensagem "Cadastro realizado com sucesso" no app.

---

## 10. Contatos

| Ação | Endpoint | Uso |
|---|---|---|
| Listar contatos úteis (ex.: Portaria) | `GET /contatos` | App |
| Cadastrar/editar contato | `POST /contatos` / `PUT /contatos/{id}` | Portal |

**Modelo `Contato`**
```json
{
  "id": "string",
  "categoria": "Portaria",
  "whatsapp": "11987654321",
  "email": "portaria@example.com",
  "telefone": "1187654321",
  "ramal": "100"
}
```

---

## 11. Considerações de Integração

- **Sincronização em tempo real:** para reduzir polling no app (novos comunicados, status de reserva/autorização), avaliar uso de *push notifications* (FCM) disparadas pelo backend quando o portal altera um status.
- **Auditoria:** toda ação do portal que altera dados visíveis ao morador (boleto, reserva, autorização, mudança) deve registrar `alteradoPor` e `alteradoEm` para rastreabilidade.
- **Permissões:** endpoints de escrita usados pelo portal (`POST/PUT/PATCH/DELETE` em boletos, comunicados, lançamentos) exigem `role = PORTARIA` ou `ADMIN`; o backend deve retornar `403 Forbidden` caso um token de morador tente acessá-los.
- **Versionamento:** o prefixo `/v1` permite introduzir `/v2` futuramente sem quebrar clientes já publicados.

---

## 12. Próximos passos sugeridos

1. Validar este contrato de API com quem for implementar o backend.
2. Definir o modelo de dados (banco) a partir das entidades listadas.
3. Especificar os eventos de notificação push por módulo.
