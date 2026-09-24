# Chatbot de Consulta Meteorológica no Telegram via n8n

Solução de automação backend para consulta de condições meteorológicas em tempo real no Telegram, construída sobre o ecossistema n8n implantado em modo de fila (Queue Mode) com suporte de infraestrutura containerizada via Docker e Portainer.

---

## Arquitetura da Solução

O projeto utiliza uma arquitetura de microsserviços containerizada para garantir alta disponibilidade, isolamento e persistência de dados.

- Telegram Client -> Ngrok Tunnel -> n8n Editor / Worker
- OpenWeather API / PostgreSQL / Redis

### Componentes de Infraestrutura

- **n8n (Queue Mode):** Motor de orquestração dividido entre nó de edição (editor) e nós de processamento (workers).
- **PostgreSQL:** Banco de dados relacional para armazenamento de estados, execuções e metadados dos fluxos.
- **Redis:** In-memory data store responsável pelo gerenciamento da fila de tarefas entre o editor e os workers.
- **Ngrok:** Proxy reverso para exposição do webhook local via HTTPS com endpoint estático.
- **Portainer:** Interface de gerenciamento e monitoramento da stack de containers Docker.

---

## Fluxo de Execução do Workflow

1. **Gatilho de Entrada (Telegram Trigger):** Captura interações recebidas via Webhook da API de Bots do Telegram.
2. **Triagem de Comando (If):**
   - **Fluxo /start:** Identifica comandos de inicialização e dispara uma mensagem com instruções de uso.
   - **Fluxo Padrão:** Encaminha mensagens de texto genéricas para o pipeline de consulta de clima.
3. **Normalização de Dados (Edit Fields):** Sanitização do texto digitado pelo usuário, aplicando tratamento de caracteres para compatibilidade com o padrão aceito pela API externa.
4. **Integração de Clima (HTTP Request):** Requisição à API do OpenWeather enviando os parâmetros de localização sanitizados com restrição geográfica (,BR) e unidade métrica (metric).
5. **Tratamento de Exceções e Resposta:**
   - **Rota de Sucesso:** Formatação e envio dos dados de temperatura coletados.
   - **Rota de Erro:** Captura de falhas HTTP (cidades não localizadas ou entradas inválidas) com retorno amigável ao usuário.

---

## Instalação e Configuração

### Pré-requisitos

- Docker e Docker Compose configurados no ambiente host.
- Bot criado no Telegram via @BotFather com o HTTP API Token.
- Chave de API (API Key) ativa na plataforma OpenWeather.
- Domínio reservado no Ngrok para recepção de webhooks.

### Configuração do Ambiente

1. Clone este repositório para o servidor local:

```bash
git clone [https://github.com/usuario/meu-repositorio.git](https://github.com/usuario/meu-repositorio.git)
cd meu-repositorio
```bash
git clone [https://github.com/usuario/meu-repositorio.git](https://github.com/usuario/meu-repositorio.git)
cd meu-repositorio
