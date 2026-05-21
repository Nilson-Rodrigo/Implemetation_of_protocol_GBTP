# Gabio Bank — Cliente Web (gabio-client)

> Implementação do protocolo **GBTP** (Gabio Bank Transaction Protocol) — cliente web em HTML + TypeScript.
> Disciplina de Redes de Computadores · Entrega: 04/06/2025

---

## Sumário

1. [Visão Geral do Projeto](#visão-geral-do-projeto)
2. [Estrutura de Pastas](#estrutura-de-pastas)
3. [Instalação e Execução](#instalação-e-execução)
4. [Protocolo GBTP](#protocolo-gbtp)
   - [Formato das Requisições](#formato-das-requisições)
   - [Formato das Respostas](#formato-das-respostas)
   - [Operações Suportadas](#operações-suportadas)
5. [Módulo de Serialização e Parser](#módulo-de-serialização-e-parser)
   - [serializeRequest](#serializerequest)
   - [parseResponse](#parseresponse)
   - [Utilitários](#utilitários)
6. [Validações do Cliente](#validações-do-cliente)
7. [Exemplos de Uso](#exemplos-de-uso)
8. [Integrantes do Grupo](#integrantes-do-grupo)

---

## Visão Geral do Projeto

O **gabio-client** é a interface web que se comunica com o servidor `gabio-server` via **WebSocket**, usando mensagens formatadas segundo o protocolo textual **GBTP**. O usuário pode:

- Consultar o saldo de uma conta (`BALANCE`)
- Realizar depósitos (`DEPOSIT`)
- Efetuar saques (`WITHDRAW`)
- Transferir valores entre contas (`TRANSFER`)

A comunicação é **totalmente textual**, inspirada no protocolo CNET: cada mensagem é uma sequência de pares `CHAVE:VALOR` separados por `\n`.

---

## Estrutura de Pastas

```
gabio-client/
├── src/
│   ├── gbtp-protocol.ts   # Serialização, parsing e validação do protocolo (Wesley)
│   ├── App.tsx            # Componente principal do formulário (Ana)
│   └── main.ts            # Inicialização e conexão WebSocket
├── index.html
├── package.json
├── tsconfig.json
└── README.md              # Este arquivo (Wesley)
```

---

## Instalação e Execução

### Pré-requisitos

- [Node.js](https://nodejs.org/) v18+
- npm v9+

### Passos

```bash
# 1. Clone o repositório
git clone <url-do-repositório>
cd gabio-client

# 2. Instale as dependências
npm install

# 3. Inicie o cliente em modo de desenvolvimento
npm start
```

O cliente abrirá em `http://localhost:5173` (ou porta configurada no Vite).

> **Atenção:** o servidor `gabio-server` precisa estar rodando em `ws://localhost:8080` para que o cliente consiga se conectar. Consulte o README do servidor para instruções de execução.

---

## Protocolo GBTP

O GBTP é um protocolo textual de camada de aplicação. As mensagens são strings simples compostas por pares `CHAVE:VALOR` separados por nova linha (`\n`).

### Formato das Requisições

Toda requisição enviada pelo cliente tem **exatamente 4 linhas**, na seguinte ordem:

```
OPERATION:<operação>
ACCOUNT_ID:<id da conta de origem>
TO_ACCOUNT_ID:<id da conta de destino ou vazio>
VALUE:<valor numérico com 2 casas decimais>
```

| Campo          | Obrigatório | Descrição                                                             |
|----------------|-------------|-----------------------------------------------------------------------|
| `OPERATION`    | Sempre      | Uma de: `BALANCE`, `DEPOSIT`, `WITHDRAW`, `TRANSFER`                  |
| `ACCOUNT_ID`   | Sempre      | Identificador da conta principal (origem)                             |
| `TO_ACCOUNT_ID`| Só TRANSFER | Identificador da conta de destino; **vazio** nas demais operações     |
| `VALUE`        | Sempre      | Valor da transação. Use `0.00` para `BALANCE`                         |

### Formato das Respostas

Toda resposta do servidor tem **exatamente 3 linhas**:

```
STATUS:<OK|ERROR>
MESSAGE:<mensagem descritiva>
BALANCE:<saldo atual com 2 casas decimais>
```

| Campo     | Descrição                                                        |
|-----------|------------------------------------------------------------------|
| `STATUS`  | `OK` se a operação foi bem-sucedida; `ERROR` caso contrário      |
| `MESSAGE` | Texto explicativo retornado pelo servidor                        |
| `BALANCE` | Saldo atual da conta principal após a operação (quando aplicável)|

### Operações Suportadas

#### BALANCE — Consulta de Saldo

```
# Requisição
OPERATION:BALANCE
ACCOUNT_ID:1234
TO_ACCOUNT_ID:
VALUE:0.00

# Resposta
STATUS:OK
MESSAGE:Saldo consultado com sucesso
BALANCE:250.00
```

#### DEPOSIT — Depósito

```
# Requisição
OPERATION:DEPOSIT
ACCOUNT_ID:1234
TO_ACCOUNT_ID:
VALUE:100.00

# Resposta
STATUS:OK
MESSAGE:Depósito realizado com sucesso
BALANCE:350.00
```

#### WITHDRAW — Saque

```
# Requisição
OPERATION:WITHDRAW
ACCOUNT_ID:1234
TO_ACCOUNT_ID:
VALUE:50.00

# Resposta (sucesso)
STATUS:OK
MESSAGE:Saque efetuado
BALANCE:300.00

# Resposta (saldo insuficiente)
STATUS:ERROR
MESSAGE:Saldo insuficiente
BALANCE:30.00
```

#### TRANSFER — Transferência

```
# Requisição
OPERATION:TRANSFER
ACCOUNT_ID:1234
TO_ACCOUNT_ID:5678
VALUE:75.00

# Resposta (sucesso)
STATUS:OK
MESSAGE:Transferência concluída
BALANCE:225.00

# Resposta (conta destino inexistente)
STATUS:ERROR
MESSAGE:Conta de destino inexistente
BALANCE:225.00
```

---

## Módulo de Serialização e Parser

O arquivo `src/gbtp-protocol.ts` centraliza toda a lógica de comunicação com o protocolo. Ele expõe funções puras e tipadas, sem dependências externas.

### `serializeRequest`

Converte os dados do formulário (objeto `GBTPRequest`) na string de protocolo exata, pronta para envio via WebSocket.

```typescript
import { serializeRequest } from "./gbtp-protocol";

const mensagem = serializeRequest({
  operation: "DEPOSIT",
  accountId: "1234",
  value: 100,
});

console.log(mensagem);
// OPERATION:DEPOSIT
// ACCOUNT_ID:1234
// TO_ACCOUNT_ID:
// VALUE:100.00
```

**Assinatura:**

```typescript
function serializeRequest(req: GBTPRequest): string
```

**Parâmetros — interface `GBTPRequest`:**

| Propriedade   | Tipo             | Descrição                                  |
|---------------|------------------|--------------------------------------------|
| `operation`   | `GBTPOperation`  | `"BALANCE"` \| `"DEPOSIT"` \| `"WITHDRAW"` \| `"TRANSFER"` |
| `accountId`   | `string`         | ID da conta de origem                      |
| `toAccountId` | `string?`        | ID da conta de destino (apenas TRANSFER)   |
| `value`       | `number`         | Valor da transação                         |

**Lança** `GBTPValidationError` se os dados forem inválidos (ver [Validações](#validações-do-cliente)).

---

### `parseResponse`

Recebe a string bruta retornada pelo servidor e extrai `STATUS`, `MESSAGE` e `BALANCE` em um objeto tipado `GBTPResponse`.

```typescript
import { parseResponse } from "./gbtp-protocol";

const resposta = parseResponse(
  "STATUS:OK\nMESSAGE:Saldo consultado com sucesso\nBALANCE:250.00"
);

console.log(resposta);
// { status: "OK", message: "Saldo consultado com sucesso", balance: 250 }
```

**Assinatura:**

```typescript
function parseResponse(raw: string): GBTPResponse
```

**Retorno — interface `GBTPResponse`:**

| Propriedade | Tipo               | Descrição                                         |
|-------------|--------------------|----------------------------------------------------|
| `status`    | `"OK" \| "ERROR"`  | Resultado da operação                              |
| `message`   | `string`           | Mensagem descritiva do servidor                    |
| `balance`   | `number \| null`   | Saldo atual; `null` se ausente na resposta          |

**Lança** `GBTPFormatError` se a resposta for malformada ou campos obrigatórios estiverem ausentes.

---

### Utilitários

| Função           | Descrição                                                         |
|------------------|-------------------------------------------------------------------|
| `isSuccess(res)` | Retorna `true` se `res.status === "OK"`                           |
| `formatBalance(balance)` | Formata o saldo em BRL: `1234.5` → `"R$ 1.234,50"` / `null` → `"—"` |

---

## Validações do Cliente

As seguintes validações são feitas por `serializeRequest` **antes** de enviar qualquer mensagem ao servidor:

| Regra                                                                 | Operações afetadas         |
|-----------------------------------------------------------------------|----------------------------|
| `ACCOUNT_ID` não pode ser vazio                                       | Todas                      |
| `VALUE` não pode ser negativo                                         | Todas                      |
| `VALUE` deve ser `0` para `BALANCE`                                   | BALANCE                    |
| `VALUE` deve ser maior que `0`                                        | DEPOSIT, WITHDRAW, TRANSFER|
| `TO_ACCOUNT_ID` é obrigatório e não pode ser vazio                    | TRANSFER                   |
| `ACCOUNT_ID` e `TO_ACCOUNT_ID` não podem ser iguais                  | TRANSFER                   |

---

## Exemplos de Uso

### Conexão e envio via WebSocket

```typescript
import { serializeRequest, parseResponse, isSuccess, formatBalance } from "./gbtp-protocol";

const ws = new WebSocket("ws://localhost:8080");

ws.onopen = () => {
  // Montar e enviar uma requisição de saque
  const msg = serializeRequest({
    operation: "WITHDRAW",
    accountId: "1001",
    value: 50,
  });

  ws.send(msg);
};

ws.onmessage = (event) => {
  const resposta = parseResponse(event.data as string);

  if (isSuccess(resposta)) {
    console.log("Operação bem-sucedida!");
    console.log("Novo saldo:", formatBalance(resposta.balance));
  } else {
    console.error("Erro:", resposta.message);
  }
};
```

---

## Integrantes do Grupo

| Nome      | Responsabilidade                                              |
|-----------|---------------------------------------------------------------|
| **Ana**   | HTML/CSS e Interface Inicial   |
| **Wesley**| Módulo de Parsing do GBTP (`src/protocol.ts`) e Documentação(README.MD) |
| **Nilson**| Conexão WebSocket e Integração (`src/main.ts`)                                      |
