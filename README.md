### Visão Geral

MVP de um aplicativo de relacionamento esportivo com **descoberta por proximidade (opt-in)** e **coincidência de janelas de lugar/horário**. Dados persistentes ficam no Postgres (via JPA). **Presença ao vivo** é efêmera (cache in-memory no MVP, Redis GEO em Evolução).

### Princípios

* **Simplicidade primeiro**: menos tabelas, menos relacionamentos, sem features supérfluas.
* **Privacidade por padrão**: minimizar PII, nada de trilhas GPS; presença com TTL.
* **Segurança básica**: chat só após match; bloqueio e denúncia em qualquer ponto; consentimento registrado.

### Entidades (resumo e justificativa)

* **users**: identidade/autenticação e status (enabled/locked/role).
* **user_profiles**: apresentação do usuário e **sports_json** (ex.: `["RUN","GYM","SURF"]`).
* **places**: catálogo de locais (academia, pista etc.) para padronizar referência e facilitar coincidência.
* **user_place_windows**: **opt-in espaço/tempo** (local + dia da semana + faixa horária + visibilidade). Substitui rastreio ao vivo.
* **likes** → **matches** → **messages**: fluxo social mínimo (like unilateral, match bilateral, chat pós-match).
* **reports** e **blocks**: segurança e moderação.
* **consent_logs**: trilha de consentimento (LGPD).

### Proximidade (não persistente)

* **MVP**: mapa in-memory com TTL (ex.: 10–15 min) atualizado somente quando a tela “Descobrir” está em uso (pull + refresh manual).
* **Evolução**: Redis GEO + TTL (GEOADD/GEOSEARCH + SETEX). Não guardar histórico.

### Cardinalidades

* `users 1–1 user_profiles`
* `users 1–N user_place_windows`, `places 1–N user_place_windows`
* `users 1–N likes` (papéis from/to)
* `users N–N users` via `matches` (materializado)
* `matches 1–N messages`
* `users 1–N reports` (papéis reporter/reported); `reports (opt) N–1 messages`
* `users 1–N blocks` (papéis blocker/blocked)
* `users 1–N consent_logs`

### Decisões de simplificação

* **`sports_json`** em `user_profiles` (STRING/JSON) em vez de N:N `UserSport`.
* Relacionamentos **unidirecionais** no JPA para reduzir complexidade.
* Sem PostGIS; sem coleções mapeadas complexas; sem converters custom.
* Chat pode começar com **long polling** (ou WebSocket simples).

### Regras de Negócio (essenciais)

* **Descoberta híbrida** (adotada):

  1. interseção de `sports`,
  2. proximidade ao vivo (recíproca, opt-in),
  3. coincidência de janelas,
  4. exploração leve.
* **Match e chat**: chat só após match; bloquear encerra e impede reexibição.
* **Privacidade**: não expor lat/lng de terceiros; apenas buckets ou distância aproximada.
* **Consentimento**: log a cada aceite/alteração de escopo.

### Migração futura

* Adicionar novos esportes → migrar `sports_json` para tabela de relacionamento `UserSport` (N:N).
* Proximidade → trocar in-memory por Redis GEO (sem mudar APIs).
* Chat → escalar para WebSocket dedicado ou serviço separado.
* Observabilidade → métricas (p99 de descoberta/chat), auditoria avançada, retentativas.

### Itens LGPD

* **Minimização**: sem telefone/redes sociais; sem localização histórica.
* **Transparência**: tela clara de opt-in; endpoint “apagar meus dados”.
* **Segurança**: hash de senha (bcrypt/argon2), HTTPS, rate-limit; logs sem PII sensível.

### Integrantes do grupo

* César Martins dos Santos Borba |   `cesarmartins350@gmail.com`
* Renan Oliveira Ewbank |   `renanewbank@gmail.com`
* Vitor Castro Dias |   `vitor.castro.diass@gmail.com`
