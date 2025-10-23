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

## Arquitetura e Implantação em Produção

Esta seção descreve a arquitetura do provider e como implantá-lo em um ambiente de produção usando Docker Swarm, Traefik e Portainer.

### Arquitetura Simplificada

Para facilitar o entendimento, podemos comparar os componentes do sistema com analogias do dia a dia:

- **Provider WhatsMeow (Este Serviço):** Pense nele como o "Atendente de WhatsApp". Ele conecta uma conta do WhatsApp ao sistema e é responsável por enviar e receber todas as mensagens e mídias.

- **RabbitMQ (O Carteiro Digital):** O RabbitMQ atua como um sistema de correio inteligente. Quando o UnoAPI precisa enviar uma mensagem para um cliente, ele não entrega diretamente ao Atendente; ele a coloca em uma "caixa de correio" (uma fila) no RabbitMQ. O Atendente (provider) então retira a mensagem dessa fila para enviar ao destinatário final via WhatsApp. Da mesma forma, quando uma mensagem é recebida do WhatsApp, o Atendente a entrega ao RabbitMQ, que a encaminha de volta para o UnoAPI. Isso garante que nenhuma mensagem seja perdida, mesmo que o Atendente esteja ocupado ou reiniciando.

- **Redis (O Quadro de Status):** O Redis funciona como um quadro de avisos público onde cada "Atendente" (sessão do WhatsApp) anota seu status atual: `online`, `offline`, `conectando`, etc. O UnoAPI pode consultar rapidamente este quadro para saber a situação de cada número conectado. Isso é muito mais eficiente do que perguntar a cada Atendente individualmente.

### Implantação em Produção com Docker Swarm

Ambientes de produção exigem alta disponibilidade e escalabilidade. O Docker Swarm é uma excelente opção para orquestrar contêineres, enquanto o Traefik atua como um roteador inteligente (proxy reverso) e o Portainer oferece uma interface gráfica para gerenciamento.

A seguir, um guia passo a passo para implantar o provider em um cluster Docker Swarm.

#### Pré-requisitos

- Um cluster Docker Swarm com pelo menos um nó manager.
- Traefik configurado como proxy reverso para o cluster.
- Portainer instalado e gerenciando o cluster (opcional, mas recomendado).

#### Passo 1: Criar o Arquivo de Stack

Em vez de usar um `docker-compose.yml` simples, em produção com Swarm, usamos um arquivo de "stack" para definir todos os serviços, redes e volumes necessários.

Crie um arquivo chamado `unoapi-stack.yml` com o seguinte conteúdo:

```yaml
version: '3.9'

services:
  whatsmeow-adapter:
    image: ghcr.io/mbap-dev/provider-whatsmeow:latest
    networks:
      - unoapi-net
    volumes:
      - whatsmeow-data:/var/lib/whatsmeow
    environment:
      # --- Conexões Essenciais ---
      # Aponta para o serviço RabbitMQ dentro da mesma stack
      AMQP_URL: "amqp://user_test:123456@rabbitmq:5672/vhost"
      # Aponta para o serviço Redis para registro de status
      REDIS_URL: "redis://redis:6379/0"

      # --- Configurações do Provider ---
      HTTP_ADDR: ":8080"
      SESSION_STORE: "/var/lib/whatsmeow"
      LOG_LEVEL: "info" # Use "debug" para mais detalhes, "error" para menos
      REJECT_CALLS: "true"
      REJECT_CALLS_MESSAGE: "No momento, não podemos atender chamadas. Por favor, envie uma mensagem de texto."

    deploy:
      mode: replicated
      replicas: 1 # Comece com 1 e aumente conforme a necessidade
      placement:
        constraints:
          - node.role == worker # Recomendado rodar em nós workers
      restart_policy:
        condition: on-failure
      labels:
        # --- Roteamento com Traefik ---
        # Habilita o Traefik para este serviço
        - "traefik.enable=true"
        # Define o roteador para o tráfego HTTP
        - "traefik.http.routers.whatsmeow.rule=Host(`whatsapp.sua-empresa.com`)" # Mude para seu domínio
        - "traefik.http.routers.whatsmeow.entrypoints=websecure"
        - "traefik.http.services.whatsmeow.loadbalancer.server.port=8080"
        - "traefik.docker.network=traefik-public" # Rede pública do Traefik

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
          - node.role == manager # Bom lugar para o broker

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
  traefik-public: # Rede externa para o Traefik
    external: true

volumes:
  whatsmeow-data:
  rabbitmq-data:
  redis-data:
```

**Observações Importantes:**

- **Domínio:** Altere `whatsapp.sua-empresa.com` para o domínio ou subdomínio que você apontou para o seu cluster Swarm.
- **Rede Traefik:** O nome `traefik-public` deve corresponder ao nome da rede que seu serviço Traefik usa para se conectar a outros contêineres.
- **Credenciais:** Em um ambiente de produção real, use "Docker Secrets" para gerenciar as senhas do RabbitMQ em vez de passá-las diretamente no ambiente.

#### Passo 2: Implantar a Stack

Você pode implantar a stack via linha de comando ou pelo Portainer.

**Via Linha de Comando:**

1.  Conecte-se ao seu nó manager do Swarm via SSH.
2.  Copie o conteúdo do arquivo `unoapi-stack.yml` para o servidor.
3.  Execute o comando:
    ```bash
    docker stack deploy -c unoapi-stack.yml unoapi
    ```

**Via Portainer:**

1.  No menu à esquerda, clique em "Stacks".
2.  Clique em "Add stack".
3.  Dê um nome para a stack (ex: `unoapi`).
4.  Cole o conteúdo do `unoapi-stack.yml` no editor web.
5.  Clique em "Deploy the stack".

#### Passo 3: Verificar a Implantação

Após alguns instantes, os serviços estarão rodando. Você pode verificar de várias formas:

- **No Portainer:** Vá para a seção "Services" e você verá `unoapi_whatsmeow-adapter`, `unoapi_rabbitmq` e `unoapi_redis`. Todos devem ter o status "running".
- **Via Linha de Comando:** Execute `docker stack ps unoapi` para ver o status de cada tarefa.
- **Acessando a API:** Tente acessar o endpoint de health check através do domínio que você configurou (ex: `https://whatsapp.sua-empresa.com/healthz`). Você deve receber uma resposta `OK`.
- **Painel do RabbitMQ:** Para visualizar as filas e mensagens, você pode expor a porta `15672` do RabbitMQ através do Traefik, adicionando as labels apropriadas ao serviço `rabbitmq`.

Com esses passos, o provider estará rodando em um ambiente robusto e pronto para produção.
