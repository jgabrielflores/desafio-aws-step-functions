# AWS Step Functions — Processador de Pedidos

Projeto desenvolvido como parte do desafio da formação **AWS Cloud Foundations** da [DIO](https://www.dio.me/), demonstrando na prática o uso de **AWS Step Functions** para orquestração de fluxos de trabalho serverless integrados ao **Amazon DynamoDB**.

---

## O que é AWS Step Functions?

AWS Step Functions é um serviço de **orquestração de serviços** que permite criar fluxos de trabalho visuais (workflows) conectando diferentes serviços da AWS. Funciona no modelo **low-code**: você define os estados em JSON (ASL), e o console gera o grafo visual automaticamente.

**Principais conceitos:**

| Conceito | Descrição |
|---|---|
| **State Machine** | O workflow completo, composto por estados |
| **Pass** | Transforma dados via JSONata sem chamar serviços externos |
| **Task** | Executa um serviço AWS (DynamoDB, Lambda, S3, etc.) |
| **Choice** | Ramifica o fluxo com base em uma condição |
| **Succeed / Fail** | Finaliza a execução com sucesso ou erro |
| **ASL** | Amazon States Language — JSON que define o workflow |
| **JSONata** | Query language moderna para transformar dados dentro do ASL |

---

## Caso de Uso: Processador de Pedidos

Um workflow que simula o fluxo de aprovação de pedidos de e-commerce, **persistindo o resultado no DynamoDB**:

1. **VerificarEstoque** — Calcula se há estoque (`quantidade <= 10`) via JSONata
2. **TemEstoque** — Decisão: aprovado ou recusado
3. **ProcessarPagamento** — Gera o ID de pagamento (caminho aprovado)
4. **SalvarPedido** — Persiste o pedido aprovado no DynamoDB
5. **RegistrarRecusa** — Persiste o pedido recusado no DynamoDB
6. **PedidoAprovado / PedidoRecusado** — Estado final

### Diagrama do Fluxo

```
         INPUT
           │
           ▼
  ┌─────────────────────┐
  │   VerificarEstoque  │  Pass — JSONata computa estoque_disponivel
  └──────────┬──────────┘
             │
             ▼
       ┌───────────┐
       │ TemEstoque│  Choice
       └─────┬─────┘
             │
      ┌──────┴──────┐
      │quantidade   │
      │  <= 10?     │
      └──────┬──────┘
         SIM │                    NÃO
             │                     │
             ▼                     ▼
  ┌───────────────────┐  ┌──────────────────────┐
  │ ProcessarPagamento│  │    RegistrarRecusa    │
  │ Pass — pagamento_id│  │  Task — DynamoDB     │
  └─────────┬─────────┘  │  (status = RECUSADO) │
            │            └──────────┬───────────┘
            ▼                       │
  ┌──────────────────┐              ▼
  │   SalvarPedido   │   ┌──────────────────┐
  │  Task — DynamoDB │   │  PedidoRecusado  │
  │ (status=APROVADO)│   │      Fail        │
  └─────────┬────────┘   └──────────────────┘
            │
            ▼
  ┌──────────────────┐
  │  PedidoAprovado  │
  │     Succeed      │
  └──────────────────┘
```

### Workflow Visual no Console AWS

![Workflow Visual](images/testes/01-state-machine-overview.png)

---

## Serviços AWS Utilizados

| Serviço | Uso |
|---|---|
| **AWS Step Functions** | Orquestração do workflow (Standard type) |
| **Amazon DynamoDB** | Persistência do resultado de cada pedido |
| **Amazon CloudWatch** | Logs de execução (automático) |
| **AWS IAM** | Role com permissão `dynamodb:PutItem` (gerada automaticamente) |

---

## Conceito Importante: Step Functions vs CloudFormation

> **Step Functions é um orquestrador — ele coordena recursos que já existem, não os cria.**

**Quem cria recursos automaticamente é o CloudFormation/SAM** — o serviço de infraestrutura como código da AWS. Nos exemplos do professor, ao clicar em "Implantar" e aguardar até 10 minutos, era o CloudFormation provisionando todos os recursos (S3, Lambda, tabelas, roles) antes de o Step Functions poder orquestrá-los.

| Serviço | Função |
|---|---|
| **Step Functions** | Orquestra serviços que já existem |
| **CloudFormation / SAM** | Cria e provisiona os recursos na AWS |

---

## Como Criar no Console AWS

### Pré-requisito
- Conta AWS ativa (free tier é suficiente — custo estimado: $0,00)

---

### Passo 1 — Criar a Tabela DynamoDB

> **Importante:** crie a tabela **antes** de criar a State Machine. O Step Functions não cria recursos — apenas os utiliza.

1. No console AWS, busque **DynamoDB**
2. Clique em **Criar tabela**
3. **Nome:** `Pedidos`
4. **Partition key:** `pedido_id` — tipo **String**
5. Mantenha as configurações padrão (On-demand)
6. Clique em **Criar tabela** e aguarde o status **Ativo**

---

### Passo 2 — Criar a State Machine

1. Busque **Step Functions** no console AWS
2. Clique em **Criar máquina de estado**
3. Selecione **Em branco**
4. Clique na aba **Código**, apague o conteúdo e cole o JSON do arquivo [`statemachine/order_processor.asl.json`](statemachine/order_processor.asl.json)
5. O grafo visual aparece automaticamente na aba **Design**
6. Clique em **Próximo**
7. **Nome:** `ProcessadorDePedidos`
8. **Tipo:** Standard
9. **Permissões:** selecione **Criar nova função** — a AWS detecta o DynamoDB no ASL e cria a policy automaticamente
10. Clique em **Criar máquina de estado**

---

## Executando os Cenários de Teste

### Cenário 1 — Pedido Aprovado (quantidade ≤ 10)

Clique em **Iniciar execução** e cole:

```json
{
  "pedido_id": "PED-001",
  "produto": "Notebook",
  "quantidade": 2,
  "cliente": "João Silva"
}
```

**Resultado:** todos os estados do caminho aprovado ficam verdes → `PedidoAprovado` → **SUCCEEDED**

![Execução Aprovada](images/testes/02-execucao-sucesso-grafo.png)

**Detalhe do estado `SalvarPedido`** — input e output da gravação no DynamoDB:

![Detalhe SalvarPedido](images/testes/03-execucao-sucesso-detalhe.png)

**Registro gravado no DynamoDB:**

![DynamoDB Pedido Aprovado](images/testes/04-dynamodb-pedido-aprovado.png)

---

### Cenário 2 — Pedido Recusado (quantidade > 10)

```json
{
  "pedido_id": "PED-002",
  "produto": "Smartphone",
  "quantidade": 50,
  "cliente": "Maria Santos"
}
```

**Resultado:** fluxo segue o caminho direito → `RegistrarRecusa` grava no DynamoDB → `PedidoRecusado` → **FAILED**

![Execução Recusada](images/testes/05-execucao-falha-grafo.png)

**Detalhe do estado `RegistrarRecusa`** — mostra `estoque_disponivel: false` no input:

![Detalhe RegistrarRecusa](images/testes/06-execucao-falha-detalhe.png)

---

## Resultado Final no DynamoDB

Após os dois testes, a tabela `Pedidos` contém ambos os registros:

![DynamoDB Todos os Registros](images/testes/07-dynamodb-todos-registros.png)

---

## Histórico de Execuções

O console Step Functions registra todas as execuções com status, duração e timestamp:

![Histórico de Execuções](images/testes/08-historico-execucoes.png)

---

## Conceitos Demonstrados

- **State Machine** com 7 estados encadeados
- **Pass State** com transformação de dados via **JSONata** (sem código externo)
- **Choice State** com condição booleana dinâmica calculada em runtime (`quantidade <= 10`)
- **Task State** com integração nativa ao **DynamoDB** (`putItem`)
- **Succeed / Fail** como estados terminais distintos
- **ASL (Amazon States Language)** com `QueryLanguage: JSONata`
- **Redrive** — reaproveitamento de execução a partir do estado falho
- Dois caminhos de execução produzindo resultados persistidos no banco de dados

---

## Estrutura do Repositório

```
DesafioStepFunction/
├── README.md
├── statemachine/
│   └── order_processor.asl.json   # Definição completa da State Machine
└── images/
    └── testes/
        ├── 01-state-machine-overview.png
        ├── 02-execucao-sucesso-grafo.png
        ├── 03-execucao-sucesso-detalhe.png
        ├── 04-dynamodb-pedido-aprovado.png
        ├── 05-execucao-falha-grafo.png
        ├── 06-execucao-falha-detalhe.png
        ├── 07-dynamodb-todos-registros.png
        └── 08-historico-execucoes.png
```

---

## Referências

- [AWS Step Functions — Documentação Oficial](https://docs.aws.amazon.com/step-functions/)
- [Amazon States Language](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html)
- [JSONata no Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html)
- [Integração Step Functions + DynamoDB](https://docs.aws.amazon.com/step-functions/latest/dg/dynamodb-iam.html)
- [DIO — AWS Cloud Foundations](https://www.dio.me/)
