# FacilitaPay — MVP

Plataforma que permite ao usuário somar limites de vários cartões para realizar um único pagamento à vista para o recebedor (Pix dinâmico ou link de pagamento). No backoffice, a compra é fracionada entre cartões do próprio usuário, porém o recebedor sempre recebe em 1x. Foco em simplicidade, segurança e aproveitamento de infraestrutura existente.

---

## Objetivo do projeto

* Viabilizar compras à vista para o recebedor usando múltiplos cartões do usuário.
* Reduzir complexidade: sem parcelamento para o recebedor, sem gestão de juros/antecipação do merchant.
* Lançar um MVP funcional e seguro em arquitetura MVC com banco relacional e frontend server-side (Thymeleaf) ou SPA leve.

---

## Escopo do MVP

* Recebedor sempre à vista (Pix dinâmico ou link one-shot do gateway).
* Usuário paga à vista em cada cartão, mas fracionado entre N cartões (sem juros via plataforma).
* Tudo-ou-nada: se qualquer autorização falhar, nada é capturado e o recebedor não é pago.
* Integração externa mínima: geração de Pix dinâmico ou link de pagamento via um gateway (cliente HTTP).
* Autenticação e autorização com Spring Security (JWT ou sessão) e checagem de propriedade dos recursos.

---

## Funcionalidades

### Para o usuário

* Cadastro e autenticação (signup/login).
* Gerenciamento de cartões (tokenização via gateway; a plataforma não guarda PAN).
* Criação de uma intenção de pagamento (valor total, método de saída: Pix ou Link).
* Rateio automático ou manual do valor entre os cartões cadastrados.
* Autorização “em lote” das cobranças por cartão.
* Captura das autorizações quando todas aprovarem.
* Geração e exibição do Pix dinâmico ou link one-shot (válido por janela curta).
* Acompanhamento do status da operação (linha do tempo).
* Cancelamento automático e transparente em caso de falhas parciais.

### Para o recebedor 

* Recebimento à vista por Pix dinâmico ou link do gateway.
* Comprovante simples da operação.

### Para o administrador

* Consulta de intents, filtros por status, auditoria de eventos e métricas iniciais.

---

## Regras de negócio

* O recebedor recebe sempre em 1x; a FacilitaPay não parcela o recebedor.
* O usuário pode dividir o valor total entre vários cartões, cada um cobrado à vista.
* Somente quando todas as autorizações forem aprovadas a plataforma captura e paga o recebedor.
* Se qualquer autorização falhar, a plataforma desfaz as outras autorizações (void) e encerra a intent como falha.
* Estornos devem ser simples: preferência por um único fluxo de devolução à origem (ex.: Pix de devolução após confirmação de entrada do recebedor).

---

## Arquitetura

* Padrão: MVC com Spring Boot.
* Banco de dados relacional (PostgreSQL em produção; H2 em desenvolvimento).
* JPA/Hibernate para persistência.
* Spring Security para autenticação/autorização.
* Frontend:

  * Monólito com Thymeleaf (SSR) para telas do usuário e admin, ou
  * SPA leve (React/Vue) consumindo os mesmos endpoints (opcional).
* Integração externa via adaptadores/clients HTTP (ex.: gateway de pagamento; no MVP usar implementação mock).

### Diagrama inicial

```
[ View (Thymeleaf/SPA) ]
        │
        ▼
[ Controllers (MVC) ]
        │
        ▼
[ Services / Regras ]
        │
        ├──> [ PaymentGatewayClient (Pix/Link) ]
        │
        ▼
[ Repositories (JPA) ]
        │
        ▼
[ DB relacional (PostgreSQL/H2) ]
```

---

## Modelo de dados

Relações 1-N: User → Card e PaymentIntent → CardCharge.

* User(id, name, email, passwordHash, createdAt)
* Card(id, user_id, brand, last4, token, holderName, expMonth, expYear, createdAt)
* PaymentIntent(id, user_id, amountCents, currency, receiverType[PIX|LINK], receiverPayload(json), status[CREATED|AUTHORIZING|AUTHORIZED|PAID|FAILED|CANCELED], createdAt, updatedAt)
* CardCharge(id, payment_intent_id, card_id, amountCents, status[PENDING|AUTHORIZED|CAPTURED|FAILED|CANCELED], externalAuthId, externalCaptureId, createdAt, updatedAt)
* Payout(id, payment_intent_id, method[PIX|LINK], amountCents, status[PENDING|SENT|CONFIRMED|FAILED], externalId, createdAt, updatedAt)

Estados principais:

* PaymentIntent: CREATED → AUTHORIZING → AUTHORIZED → PAID (ou FAILED/CANCELED)
* CardCharge: PENDING → AUTHORIZED → CAPTURED (ou FAILED/CANCELED)
* Payout: PENDING → SENT → CONFIRMED (ou FAILED)

---

## Fluxos principais

1. Criação da intent

* Usuário define valor e método de saída (Pix/Link).
* Sistema cria PaymentIntent (status CREATED) e define CardCharges (rateio auto/manual).

2. Autorização

* Serviço tenta autorizar todas as CardCharges.
* Se todas aprovam: intent → AUTHORIZED. Se alguma falhar: void nas demais e intent → FAILED.

3. Captura e pagamento ao recebedor

* Captura de todas as autorizações aprovadas.
* Geração do Pix dinâmico ou link one-shot; criação do Payout e confirmação.
* Intent → PAID.

4. Falhas/compensações

* Falha em qualquer etapa: reversão segura (void/cancel) e encerramento consistente.

---

## Endpoints (mínimo)

* Autenticação:

  * POST /auth/signup
  * POST /auth/login
  * GET /me

* Cartões:

  * POST /cards (salva token e metadados)
  * GET /cards
  * DELETE /cards/{id}

* Pagamentos:

  * POST /payment-intents
  * POST /payment-intents/{id}/authorize
  * POST /payment-intents/{id}/capture
  * GET /payment-intents/{id}

* Webhooks (opcional no MVP):

  * POST /webhooks/gateway

---

## Segurança

* Spring Security; autenticação por JWT ou sessão (definir no projeto).
* Regras de autorização por papel (USER/ADMIN) e por propriedade (User só acessa seus Cards/Intents).
* Não armazenar dados sensíveis de cartão; usar tokenização do gateway (PCI fora da plataforma).
* Logs estruturados, auditoria de eventos e limites de uso (rate limit).

---

## Frontend

* Thymeleaf (SSR) para:

  * Login/Signup
  * Minhas formas de pagamento (cartões)
  * Nova intenção de pagamento (valor, método, rateio)
  * Tela de autorização e resultado (QR/link, status)
  * Lista de intents e detalhes (linha do tempo)

---

## Estrutura sugerida

```
facilitapay/
 ├─ src/main/java/.../api/          # Controllers (MVC) + DTOs
 ├─ src/main/java/.../domain/       # Models (JPA) + Services (regras)
 ├─ src/main/java/.../infra/        # Repositories, Config, Security, Migrations
 ├─ src/main/java/.../adapters/     # PaymentGatewayClient (mock)
 ├─ src/main/resources/templates/   # Thymeleaf views
 ├─ src/main/resources/static/      # CSS/JS/imagens
 └─ src/test/java/...               # Testes
```



### Integrantes do grupo

* César Martins dos Santos Borba |   `cesarmartins350@gmail.com`
* Renan Oliveira Ewbank |   `renanewbank@gmail.com`
* Vitor Castro Dias |   `vitor.castro.diass@gmail.com`
