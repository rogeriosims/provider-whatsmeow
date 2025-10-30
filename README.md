# WhatsMeow Adapter (Provider UNOAPI Cloud)

Microserviço **Go** que:
- consome mensagens **AMQP/RabbitMQ** (rota `provider.whatsmeow.*`, fila `provider.whatsmeow` criada automaticamente);
- publica webhooks no exchange de bridge do UnoAPI (default `unoapi.brigde`, tipo `direct`), com routing key igual ao número da sessão (somente dígitos) e corpo `{ "payload": <cloud_api_webhook_payload> }`.
- envia via **WhatsMeow** (WhatsApp MD);
- publica **webhooks** (Cloud-like) se `WEBHOOK_BASE` estiver definido;
- expõe HTTP para gerenciar sessões e health checks.
- para mensagens com mídia, não armazena mais em S3/MinIO; o webhook inclui `direct_path`, `media_key`, `mime_type`, `sha256` e `file_length` para que o serviço consumidor faça o download e a descriptografia.
- envio de áudio: converte para OGG/Opus mono via `ffmpeg`, calcula `seconds` e `waveform` e envia como PTT por padrão.
   - controle de PTT por env: defina `AUDIO_PTT_DEFAULT=false` para desabilitar PTT por padrão (se ausente, é `true`). O campo `audio.ptt` no payload pode sobrescrever por mensagem.
- rejeita ligações recebidas automaticamente e, opcionalmente, envia uma mensagem de resposta ao chamador.
- opção para marcar como lidas as conversas assim que uma mensagem é recebida (`MARK_READ_ON_MESSAGE=true`).


> Este provider integra com [unoapi-cloud](https://github.com/mbap-dev/unoapi-cloud) usando o padrão Cloud/Graph-like de payloads.

---

## Endpoints HTTP

- `POST /sessions/{id}/connect` — inicia/recupera cliente e, se 1º login, gera QR.
- `POST /sessions/{id}/disconnect` — encerra sessão.
- `POST /sessions/{id}/reload` — reinicia a sessão.
- `GET /sessions/{id}/qr` — **base64** do PNG do QR (durante pareamento).
- `GET /healthz` — liveness (200).
- `GET /readyz` — readiness (200 quando servidor iniciou).
- `GET /sessions/{id}/resolve?to=VALUE` — resolve o destino aplicando overrides/heurística e mapeamento PN→LID; retorna `{ input, normalized_pn, pn_jid, lid_jid?, dest_jid, used_lid }`.

### Exemplos

```bash
# Conectar/abrir sessão
curl -X POST http://localhost:8080/sessions/main/connect

# Ler QR atual (base64)
curl http://localhost:8080/sessions/main/qr
## Ligações (auto-reject)

- `REJECT_CALLS` (padrão `true`): quando uma ligação é recebida, ela é automaticamente rejeitada.
- `REJECT_CALLS_MESSAGE` (opcional): texto de resposta enviado para quem ligou após a rejeição. Suporta `\n` para quebra de linha.
  - Ex.: `REJECT_CALLS_MESSAGE="Não aceitamos ligações pelo WhatsApp.\nPor favor, envie uma mensagem."`
  - Quando a mensagem é enviada, um webhook de `statuses` (status `sent`) é publicado para o UnoAPI informando o envio dessa resposta.
## Variáveis de ambiente úteis
- `AUDIO_PTT_DEFAULT` (bool, default `true`)
- `IGNORE_STATUS_BROADCAST` (bool, default `true`) — ignora mensagens de "Atualizações/Status" (status@broadcast) recebidas; defina `false` para processar essas mensagens normalmente.
- `IGNORE_NEWSLETTERS` (bool, default `true`) — ignora mensagens de canais/Newsletters (servidor `newsletter`); defina `false` para processá-las.
 
 

## Testes
- `go test ./...` roda testes unitários (helpers de envio, normalização de MIME, waveform etc.).
- O Dockerfile executa `go test ./...` no estágio de build e falha a imagem se os testes falharem.

## Dev (Check Rápido)
- Makefile incluso:
  - `make check` → `go mod tidy`, `go vet`, `go test ./...`
  - `make build` → compila binário em `bin/whatsmeow-adapter`
  - `make docker-build` → build Docker local
  - Sem pkg-config (se preferir): `make test-nopkg` / `make build-nopkg`

## Logs
- Controle por `LOG_LEVEL`: `error` (default), `info`, `debug`.
- `debug`: inclui payloads e detalhes; `info`: resumos; `error`: apenas erros.

---

## Configuração (Variáveis de Ambiente)

A configuração do provider é feita exclusivamente através de variáveis de ambiente. A seguir, uma lista completa de todas as variáveis disponíveis.

### Variáveis Essenciais

| Variável      | Descrição                                                                                               | Padrão                                         |
| :------------ | :------------------------------------------------------------------------------------------------------ | :--------------------------------------------- |
| `AMQP_URL`    | URL de conexão completa para o RabbitMQ.                                                                | `amqp://user_test:123456@localhost:5672/vhost` |
| `REDIS_URL`   | URL de conexão completa para o Redis, usado para o status da sessão. Deixe em branco para desabilitar.      | `""`                                           |
| `HTTP_ADDR`   | Endereço e porta em que a API HTTP interna ficará escutando.                                              | `:8080`                                        |

### Comportamento do Provider

| Variável                       | Descrição                                                                                                                                                             | Padrão        |
| :----------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------ |
| `LOG_LEVEL`                    | Define o nível de detalhe dos logs. Valores: `error` (mínimo), `info` (padrão), `debug` (máximo, útil para depuração).                                                      | `info`        |
| `REJECT_CALLS`                 | Se `true`, rejeita automaticamente todas as chamadas de voz e vídeo recebidas.                                                                                          | `true`        |
| `REJECT_CALLS_MESSAGE`         | Mensagem de texto opcional a ser enviada para a pessoa cuja chamada foi rejeitada.                                                                                      | `""`          |
| `SESSION_STORE`                | Caminho no contêiner onde os dados da sessão (tokens, etc.) são armazenados para persistência.                                                                            | `./state/whatsmeow` |
| `MARK_READ_ON_MESSAGE`         | Se `true`, marca as conversas como lidas assim que uma nova mensagem é recebida.                                                                                       | `false`       |
| `AUDIO_PTT_DEFAULT`            | Se `true`, envia todas as mensagens de áudio como "Push-to-Talk" (PTT), ou seja, como se tivessem sido gravadas no app.                                                    | `true`        |
| `ALWAYS_ONLINE`                | Se `true`, mantém o status da presença como "online" enviando pings periódicos para o WhatsApp.                                                                           | `false`       |
| `ALWAYS_ONLINE_INTERVAL_SECONDS` | Intervalo em segundos para o envio dos pings de presença "online".                                                                                                       | `60`          |
| `IGNORE_STATUS_BROADCAST`      | Se `true`, ignora o recebimento de mensagens de "Status" (stories) dos contatos.                                                                                         | `true`        |
| `IGNORE_NEWSLETTERS`           | Se `true`, ignora o recebimento de mensagens de canais (Newsletters).                                                                                                  | `true`        |
| `WEBHOOK_BASE`                 | URL base para onde os webhooks de eventos (como recebimento de mensagens) serão enviados (Cloud-like). Deixe em branco para desabilitar.                                   | `""`          |
| `PN_RESOLVER_URL`              | URL de um serviço HTTP externo para resolver um número de telefone para o formato correto do WhatsApp.                                                                    | `""`          |

### Configuração do RabbitMQ

| Variável              | Descrição                                                                                                                                         | Padrão                  |
| :-------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------- |
| `AMQP_EXCHANGE`       | Nome da exchange do RabbitMQ onde as mensagens de saída são publicadas.                                                                            | `unoapi.outgoing`       |
| `AMQP_QUEUE`          | Nome da fila que será criada e consumida por este serviço.                                                                                           | `outgoing.whatsmeow`    |
| `AMQP_BINDING`        | O padrão da "routing key" usado para ligar a fila à exchange.                                                                                       | `provider.whatsmeow.*`  |
| `AMQP_WEBHOOK_EXCHANGE` | A exchange para a qual os webhooks (payloads Cloud-like) serão publicados para a UnoAPI consumir.                                                  | `unoapi.brigde`         |

## Arquitetura e Implantação em Produção

Esta seção descreve a arquitetura completa do sistema UnoAPI, incluindo a aplicação principal e este provider, e como implantar todo o conjunto em um ambiente de produção usando Docker Swarm, Traefik e Portainer.

### Arquitetura Completa do Sistema

Para entender o funcionamento, é crucial diferenciar os dois principais componentes:

-   **UnoAPI Cloud (A Aplicação Principal):** Este é o cérebro do sistema. É uma aplicação Node.js que fornece a API RESTful, gerencia a lógica de negócios, armazena configurações de sessão, webhooks e templates. Ela decide **o que** enviar e **para quem**. No entanto, ela não se conecta diretamente ao WhatsApp.

-   **Provider WhatsMeow (Este Serviço):** Este é um "conector" ou "adaptador" especializado. Sua única função é ser a **ponte** entre a UnoAPI Cloud e uma conta de WhatsApp. Ele recebe ordens da UnoAPI para enviar mensagens e repassa as mensagens recebidas do WhatsApp de volta para a UnoAPI. Você pode ter vários providers para diferentes canais (WhatsApp, Instagram, etc.).

#### Como Eles se Comunicam: O Papel do RabbitMQ e Redis

-   **RabbitMQ (O Carteiro Digital):** A comunicação entre a UnoAPI Cloud e o Provider não é direta. Ela é feita através de um sistema de mensageria, o RabbitMQ.
    1.  **Envio de Mensagem:** Quando você faz uma chamada de API para a UnoAPI Cloud para enviar uma mensagem, ela não a envia diretamente. Em vez disso, ela publica a mensagem em uma fila no RabbitMQ.
    2.  **Consumo pelo Provider:** O Provider (`whatsmeow-adapter`) está constantemente "ouvindo" essa fila. Assim que uma nova mensagem aparece, ele a pega e a envia para o destinatário final via WhatsApp.
    3.  **Recebimento de Mensagem:** Quando um usuário final responde, o Provider recebe a mensagem e a publica em outra fila no RabbitMQ, que a UnoAPI Cloud consome para processar e, se necessário, disparar webhooks.

-   **Redis (O Quadro de Status e Cache):** Redis tem duas funções principais:
    1.  **Status da Sessão:** O Provider usa o Redis para registrar o status de cada conexão do WhatsApp (`online`, `offline`, `conectando`). A UnoAPI Cloud pode ler essa informação para saber o estado de cada número.
    2.  **Cache e Fila Interna:** A UnoAPI Cloud usa o Redis para gerenciar suas próprias filas de mensagens e armazenar em cache configurações e dados de sessão para acesso rápido.

### Implantação Completa com Docker Swarm

A seguir está um guia para implantar o sistema **completo** (UnoAPI Cloud + Provider) em um cluster Docker Swarm.

#### Pré-requisitos

-   Um cluster Docker Swarm com pelo menos um nó manager.
-   Traefik configurado como proxy reverso para o cluster.
-   Portainer instalado para gerenciamento (opcional, mas recomendado).

#### Passo 1: Criar o Arquivo de Stack Completo

Crie um arquivo chamado `unoapi-stack.yml`. Este arquivo define todos os serviços necessários para o sistema funcionar.

```yaml
version: '3.9'

services:
  # Cérebro do sistema: A aplicação principal da UnoAPI
  unoapi:
    image: clairton/unoapi-cloud:latest
    networks:
      - unoapi-net
    volumes:
      - unoapi-data:/home/u/app/data
    environment:
      # --- Conexões Essenciais ---
      AMQP_URL: "amqp://user_test:123456@rabbitmq:5672/vhost"
      REDIS_URL: "redis://redis:6379/0"

      # --- Configuração Base ---
      # Domínio base para URLs de mídia, etc.
      BASE_URL: "https://unoapi.sua-empresa.com"
      PORT: "9876"
      LOG_LEVEL: "warn"

    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
      restart_policy:
        condition: on-failure
      labels:
        # --- Roteamento com Traefik para a API Principal ---
        - "traefik.enable=true"
        - "traefik.http.routers.unoapi.rule=Host(`unoapi.sua-empresa.com`)"
        - "traefik.http.routers.unoapi.entrypoints=websecure"
        - "traefik.http.services.unoapi.loadbalancer.server.port=9876"
        - "traefik.docker.network=traefik-public"

  # Conector para WhatsApp: Este provider
  provider:
    image: ghcr.io/mbap-dev/provider-whatsmeow:latest
    networks:
      - unoapi-net
    volumes:
      - provider-data:/var/lib/whatsmeow
    environment:
      # --- Conexões Essenciais ---
      AMQP_URL: "amqp://user_test:123456@rabbitmq:5672/vhost"
      REDIS_URL: "redis://redis:6379/0"

      # --- Configurações do Provider ---
      HTTP_ADDR: ":8080"
      LOG_LEVEL: "info"
      REJECT_CALLS: "true"
      REJECT_CALLS_MESSAGE: "No momento, não podemos atender chamadas."

    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
      restart_policy:
        condition: on-failure
      labels:
        # --- Roteamento com Traefik para a API do Provider ---
        - "traefik.enable=true"
        - "traefik.http.routers.provider.rule=Host(`provider.sua-empresa.com`)"
        - "traefik.http.routers.provider.entrypoints=websecure"
        - "traefik.http.services.provider.loadbalancer.server.port=8080"
        - "traefik.docker.network=traefik-public"

  # Fila de Mensagens
  rabbitmq:
    image: rabbitmq:3.12-management
    networks:
      - unoapi-net
    environment:
      RABBITMQ_DEFAULT_USER: "user_test"
      RABBITMQ_DEFAULT_PASS: "123456"
      RABBITMQ_DEFAULT_VHOST: "vhost"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    deploy:
      placement:
        constraints:
          - node.role == manager

  # Cache e Status
  redis:
    image: redis:7.2-alpine
    networks:
      - unoapi-net
    volumes:
      - redis-data:/data
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  unoapi-net:
    driver: overlay
    attachable: true
  traefik-public: # Deve ser o nome da rede externa do seu Traefik
    external: true

volumes:
  unoapi-data:
  provider-data:
  rabbitmq-data:
  redis-data:
```

**Observações Importantes:**

-   **Dois Domínios:** A configuração acima expõe **dois serviços**. Você precisará de dois subdomínios:
    1.  `unoapi.sua-empresa.com`: Para interagir com a API principal da UnoAPI.
    2.  `provider.sua-empresa.com`: Para gerenciar as sessões do WhatsApp (conectar, obter QR code, etc.).
-   **Credenciais:** Para produção, use "Docker Secrets" para gerenciar as senhas do RabbitMQ.
-   **Rede Traefik:** Certifique-se de que `traefik-public` é o nome correto da rede externa do seu Traefik.

#### Passo 2: Implantar a Stack

O processo é o mesmo de antes, mas agora implantará todo o sistema.

**Via Linha de Comando:**

1.  Conecte-se ao seu nó manager do Swarm.
2.  Salve o conteúdo acima no arquivo `unoapi-stack.yml`.
3.  Execute: `docker stack deploy -c unoapi-stack.yml unoapi`

**Via Portainer:**

1.  Vá para "Stacks" -> "Add stack".
2.  Dê o nome `unoapi` e cole o conteúdo do YAML.
3.  Clique em "Deploy the stack".

#### Passo 3: Verificar a Implantação

-   **Serviços:** No Portainer ou com `docker stack ps unoapi`, você deve ver todos os quatro serviços (`unoapi_unoapi`, `unoapi_provider`, `unoapi_rabbitmq`, `unoapi_redis`) rodando.
-   **Acessando as APIs:**
    -   Verifique a API principal: `https://unoapi.sua-empresa.com/ping` (deve retornar `pong!`).
    -   Verifique a API do provider: `https://provider.sua-empresa.com/healthz` (deve retornar `OK`).

Com esta nova configuração, você terá o ecossistema completo da UnoAPI funcionando em seu ambiente de produção.
