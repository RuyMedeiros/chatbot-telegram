# Chatbot de Consulta Meteorológica no Telegram via n8n

Solução de automação backend para consulta de condições meteorológicas em tempo real no Telegram, construída sobre o ecossistema **n8n**, implantado em modo de fila (*Queue Mode*), com suporte de infraestrutura containerizada gerenciada via **Portainer** e **Docker**.

---

## 🛠️ Arquitetura da Solução

O projeto utiliza uma arquitetura de microsserviços containerizada para garantir isolamento, persistência de dados e processamento assíncrono.

```text
[ Telegram Client ]
        │
        ▼
[ Ngrok Tunnel ]
        │
        ▼
[ n8n Editor / Worker ]
        │
   ┌────┴───────────────┐
   ▼                    ▼
[ OpenWeather API ]  [ PostgreSQL / Redis ]
```

### Componentes de Infraestrutura

- **n8n Editor (`n8n-editor-ruy`)**: interface gráfica e orquestrador principal na porta `5678`.
- **n8n Worker (`n8n-worker`)**: processador em segundo plano, executando em modo de fila (*Queue Mode*).
- **PostgreSQL (`ankane/pgvector`)**: banco de dados relacional com suporte a vetores para armazenamento das informações do n8n.
- **Redis (`redis:alpine`)**: broker *in-memory* utilizado pelo Bull para gerenciamento da fila de tarefas entre editor e worker.
- **Ngrok (`ngrok/ngrok:latest`)**: proxy reverso para exposição dos webhooks do n8n.
- **Portainer**: interface gráfica para deploy, gerenciamento de stacks e monitoramento dos containers.
- **OpenWeather API**: serviço externo utilizado para obtenção das condições meteorológicas.

> **⚠️ Segurança:** nunca publique tokens, chaves de API ou `NGROK_AUTHTOKEN` reais no GitHub. Utilize variáveis de ambiente, `.env`, Docker Secrets ou os mecanismos de credenciais do n8n. Se alguma credencial real tiver sido exposta anteriormente, revogue-a e gere uma nova.

---

## ⚙️ Passo a Passo Completo de Instalação e Execução

### Passo 1: Configuração da Stack no Portainer

1. Acesse o painel do seu **Portainer**.
2. Vá em **Stacks** → **Add stack**.
3. Nomeie a stack como `n8n-telegram-bot`.
4. No campo **Web editor**, cole o conteúdo do arquivo `docker-compose.yml`.

> **Importante:** substitua os valores de exemplo das variáveis sensíveis antes do deploy.

```yaml
services:
  n8n-editor:
    image: n8nio/n8n:latest
    container_name: n8n-editor-ruy
    ports:
      - 5678:5678
    volumes:
      - n8n-data:/home/node/.n8n
    environment:
      - WEBHOOK_URL=https://amino-tables-strung.ngrok-free.dev/
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
      - GERENETIC_TIMEZOE=America/Sao_Paulo
      - TZ=America/Sao_Paulo
      - N8N_RUNNERS_ENABLED=true 
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=false
      - N8N_GIT_NODE_DISABLE_BARE_REPOS=true
      ## Postgres
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_DATABASE=postgres
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=postgres
      - DB_POSTGRESDB_PASSWORD=postgres
      ## Queue Mode
      - EXECUTION_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis  
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true     
    networks:
      - n8n-ruy
    depends_on:
      - postgres
      - redis
    restart: always
  
  n8n-worker:
    image: n8nio/n8n:latest
    command: worker
    ports:
      - 5679:5678
    volumes:
      - n8n-data:/home/node/.n8n
    environment:
      - WEBHOOK_URL=https://amino-tables-strung.ngrok-free.dev/
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
      - GERENETIC_TIMEZOE=America/Sao_Paulo
      - TZ=America/Sao_Paulo
      - N8N_RUNNERS_ENABLED=true 
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=false
      - N8N_GIT_NODE_DISABLE_BARE_REPOS=true
      ## Postgres
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_DATABASE=postgres
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_USER=postgres
      - DB_POSTGRESDB_PASSWORD=postgres
      ## Queue Mode
      - EXECUTION_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis   
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true 
    networks:
      - n8n-ruy
    depends_on:
      - postgres
      - redis
    restart: always

  postgres:
    image: ankane/pgvector
    ports:
    - 5432:5432
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
    - POSTGRES_PASSWORD=postgres  
    - POSTGRES_DB=postgres 
    - POSTGRES_USER=postgres  
    networks:
    - n8n-ruy 
    restart: always
  
  redis:
    image: redis:alpine
    ports:
      - 6379:6379
    volumes:
      - redis-data:/data
    networks:
      - n8n-ruy
    restart: always
  
  ngrok:
    image: ngrok/ngrok:latest
    ports: 
      - 4040:4040
    volumes: 
      - ngrok-data:/etc
    command: http --domain=amino-tables-strung.ngrok-free.dev n8n-editor-ruy:5678
    environment:
      - NGROK_AUTHTOKEN= ''NGROK_AUTHTOKEN''
    restart: always
    networks:
      - n8n-ruy
    depends_on:
      - n8n-editor

networks:
  n8n-ruy:
    driver: bridge

volumes:
  n8n-data:
  postgres-data:
  redis-data:
  ngrok-data:
```

5. Clique em **Deploy the stack**.
6. Aguarde até que os containers estejam com o status `running`.

### Passo 2: Importar o Workflow no n8n

1. Acesse o n8n pelo navegador em `http://localhost:5678` ou pela URL pública configurada no Ngrok.
2. No menu superior direito do canvas, clique no ícone de três pontos (`...`).
3. Selecione **Import from file...**.
4. Escolha o arquivo `workflow-chatbot-telegram.json` presente neste repositório.

### Passo 3: Configurar a Credencial do Telegram (`TELEGRAM_BOT_TOKEN`)

1. No menu lateral esquerdo do n8n, acesse **Credentials** → **Add Credential**.
2. Pesquise por **Telegram API** e selecione a credencial correspondente.
3. No campo **Access Token**, utilize a expressão:

```text
={{ $env.TELEGRAM_BOT_TOKEN }}
```

   ou informe diretamente o token obtido por meio do **@BotFather**.
4. Salve a credencial com o nome `Telegram account`.
5. Abra o nó **Telegram Trigger** e os nós **Send a text message**.
6. No campo **Credential to connect with**, selecione a credencial configurada.

### Passo 4: Configurar a Credencial do OpenWeather (`OPENWEATHER_API_KEY`)

1. No canvas do workflow, abra o nó **HTTP Request** responsável pela consulta meteorológica.
2. Na seção **Query Parameters**, configure o parâmetro `appid` com:

```text
={{ $env.OPENWEATHER_API_KEY }}
```

   ou informe diretamente sua chave da OpenWeather.
3. Verifique se os demais parâmetros estão configurados assim:

| Parâmetro | Valor |
|---|---|
| `q` | `={{ $json.queue }},BR` |
| `units` | `metric` |

4. Salve as alterações e retorne ao canvas.

### Passo 5: Publicar e Ativar o Workflow

1. No canto superior direito da tela do n8n, clique em **Publish**.
2. O workflow ficará publicado/ativo conforme a versão do n8n utilizada.
3. A partir desse momento, o fluxo poderá receber eventos do Telegram por meio do webhook e processá-los continuamente em segundo plano.

---

## 🧪 Como Executar e Validar o Chatbot

Abra o Telegram, procure pelo seu bot e execute os testes abaixo.

### Teste 1: Comando de Boas-vindas (`/start`)

**Entrada enviada:**

```text
/start
```

**Retorno esperado:**

> 👋 Olá! Envie o nome de uma cidade para consultar o clima (ex.: São Paulo, SP).

### Teste 2: Consulta de Cidade Válida

**Entrada enviada:**

```text
São Paulo, SP
```

ou:

```text
Belém, PA
```

**Retorno esperado:**

> 🌤️ A temperatura em São Paulo é de 23°C

> *A temperatura exibida é apenas um exemplo. O valor real dependerá das condições meteorológicas retornadas pela OpenWeather no momento da consulta.*

### Teste 3: Tratamento de Erros e Cidades Inexistentes

**Entrada enviada:**

```text
CidadeInexistente123
```

**Retorno esperado:**

> ❌ Cidade não encontrada. Use o formato Cidade,UF (ex.: São Paulo,SP).

---

## 📁 Estrutura Sugerida do Repositório

```text
.
├── README.md
├── docker-compose.yml
└── workflow-chatbot-telegram.json
```

---

## 🔐 Boas Práticas de Segurança

Antes de publicar o projeto no GitHub:

- Não coloque tokens do Telegram diretamente no código.
- Não publique chaves da OpenWeather.
- Não publique seu `NGROK_AUTHTOKEN`.
- Evite versionar arquivos `.env` contendo credenciais.
- Adicione `.env` ao `.gitignore`.
- Caso uma credencial tenha sido publicada acidentalmente, revogue-a e gere uma nova imediatamente.

Exemplo de `.gitignore`:

```gitignore
.env
*.env
secrets/
```

---

## 📄 Licença

Este projeto foi desenvolvido para fins acadêmicos e práticos de automação de processos.

Sinta-se à vontade para utilizar, estudar e adaptar o código conforme necessário.
