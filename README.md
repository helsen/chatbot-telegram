# 🌤️ Chatbot de Clima no Telegram com n8n + OpenWeather

Chatbot do Telegram construído no **n8n** que recebe o nome de uma cidade, consulta a **API OpenWeather** e responde com a temperatura atual:

```
🌤️ A temperatura em Belo Horizonte é de 25°C.
```

Se a cidade não existir ou a API responder com erro:

```
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## 📁 Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `workflow-chatbot-telegram.json` | Workflow exportado do n8n (sem credenciais) |
| `workflow-telegram-chatbot.json` | Cópia idêntica (o enunciado cita os dois nomes) |
| `README.md` | Esta documentação |
| `docker-compose.yml` | (Opcional) sobe o n8n localmente |
| `.env.example` | Modelo das variáveis de ambiente (sem valores reais) |

## 🔄 Fluxo do workflow

```
Telegram Trigger
   → Formatar Entrada (Set: variável `queue`)
   → OpenWeather - Consultar Clima (HTTP Request)
   → Resposta Válida? (IF)
        ├─ true  → Formatar Mensagem (Fallback) → Google Gemini (opcional, desativado)
        │          → Mensagem Final → Enviar Temperatura (Telegram)
        └─ false → Enviar Erro (Telegram)
```

1. **Telegram Trigger** – recebe as mensagens de texto enviadas ao bot.
2. **Formatar Entrada (Set)** – cria a variável `queue` com o texto normalizado: remove espaços extras (inclusive ao redor das vírgulas), remove acentos e converte para minúsculas. Ex.: `  São Paulo , SP , BR ` → `sao paulo,sp,br`. Também guarda o `chat_id`.
3. **OpenWeather - Consultar Clima (HTTP Request)** – `GET https://api.openweathermap.org/data/2.5/weather` com *Send Query Parameters* ativado:
   - `q` = `{{ $json.queue }}` (parâmetro que a API da OpenWeather realmente lê)
   - `queue` = `{{ $json.queue }}` (mantido para atender literalmente ao enunciado; a API ignora parâmetros desconhecidos)
   - `units` = `metric` (graus Celsius)
   - `lang` = `pt_br` (português brasileiro)
   - `appid` = `{{ $env.OPENWEATHER_API_KEY }}`
   - As opções *Include Full Response* e *Never Error* estão ativas, para que o status HTTP chegue ao IF mesmo em caso de 404.
4. **Resposta Válida? (IF)** – segue pelo caminho de sucesso somente se `statusCode == 200`, `body.main.temp` existir e `body.name` não estiver vazio. Caso contrário, vai para **Enviar Erro**.
5. **Formatar Mensagem (Fallback)** – Code node que extrai a temperatura, arredonda com `Math.round` e monta a mensagem de forma determinística (`{ "message": "...", "ok": true }`).
6. **Google Gemini - Melhorar Mensagem** – opcional (ver abaixo).
7. **Mensagem Final** – usa a saída do Gemini se ela for um JSON válido e mantiver a temperatura correta; senão, usa o fallback.
8. **Enviar Temperatura / Enviar Erro** – nós *Telegram → Send Message*.

## ✅ Pré-requisitos

- n8n (testado com a versão 1.x; Docker opcional)
- Um bot do Telegram criado no [@BotFather](https://t.me/BotFather) → `TELEGRAM_BOT_TOKEN`
- Uma chave gratuita da [OpenWeather](https://home.openweathermap.org/api_keys) → `OPENWEATHER_API_KEY` (chaves novas podem levar até ~2h para ativar)
- Uma **URL pública HTTPS** para o n8n (o Telegram só entrega mensagens via webhook HTTPS). Localmente, use [ngrok](https://ngrok.com/): `ngrok http 5678`.

## 🔐 Variáveis de ambiente

| Variável | Uso |
|---|---|
| `OPENWEATHER_API_KEY` | Lida no nó HTTP Request via `{{ $env.OPENWEATHER_API_KEY }}` |
| `TELEGRAM_BOT_TOKEN` | Usada na credencial do Telegram dentro do n8n |
| `WEBHOOK_URL` | URL pública HTTPS do n8n (necessária para o Telegram Trigger) |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` | Libera o acesso a `$env` nas expressões (já definido no `docker-compose.yml`) |

> ⚠️ Nenhuma chave ou token real está neste repositório. Use `.env` (listado no `.gitignore`).

## 🐳 Subindo o n8n com Docker (opcional)

```bash
cp .env.example .env        # preencha com seus valores reais
ngrok http 5678             # copie a URL https gerada para WEBHOOK_URL no .env
docker compose up -d
```

Acesse `http://localhost:5678` e crie sua conta de owner.

Sem Docker (npm), exporte as variáveis antes de iniciar:

```bash
export OPENWEATHER_API_KEY=...
export TELEGRAM_BOT_TOKEN=...
export WEBHOOK_URL=https://seu-endereco.ngrok-free.app/
export N8N_BLOCK_ENV_ACCESS_IN_NODE=false
npx n8n
```

## 📥 Importando o workflow

1. No n8n, clique em **Workflows → Create Workflow**.
2. No menu **⋯** (canto superior direito), escolha **Import from File…** e selecione `workflow-chatbot-telegram.json`.
3. O workflow abrirá com os nós do Telegram sem credencial — configure-a conforme abaixo.

## 🔑 Configurando as credenciais

### Telegram
1. Abra o nó **Telegram Trigger** → *Credential to connect with* → **Create New Credential**.
2. No campo **Access Token**, informe a expressão `{{ $env.TELEGRAM_BOT_TOKEN }}` (ou cole o token diretamente — ele fica salvo criptografado no n8n, não no JSON exportado).
3. Salve e selecione **a mesma credencial** nos nós **Enviar Temperatura** e **Enviar Erro**.

### OpenWeather
Não há credencial a criar: o nó **OpenWeather - Consultar Clima** já usa `{{ $env.OPENWEATHER_API_KEY }}`. Basta a variável existir no ambiente do n8n e `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`.

## ▶️ Executando o chatbot

1. Salve o workflow e ative o toggle **Active** (ou clique em *Test workflow* para um teste único).
2. No Telegram, abra o seu bot e envie uma cidade no formato `Cidade,UF,BR`.

| Você envia | Resposta esperada |
|---|---|
| `Belo Horizonte,MG,BR` | 🌤️ A temperatura em Belo Horizonte é de 25°C. |
| `São Paulo,SP,BR` | 🌤️ A temperatura em São Paulo é de 22°C. |
| `rio de janeiro, rj, br` | 🌤️ A temperatura em Rio de Janeiro é de 28°C. |
| `Cidadeinexistente123` | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |

(As temperaturas variam conforme o clima do momento.)

## 🤖 Google Gemini (opcional)

- **Onde está:** nó **Google Gemini - Melhorar Mensagem**, entre *Formatar Mensagem (Fallback)* e *Mensagem Final*.
- **Estado padrão:** **desativado**. Um nó desativado apenas repassa os dados, então o fluxo usa o **fallback determinístico** — é assim que a avaliação automática roda sem custos.
- **Configuração:** temperatura `0.1`; o prompt pede resposta em português, mantendo cidade e temperatura, somente no formato `{"message":"...","ok":true}`.
- **Segurança:** o nó *Mensagem Final* só aceita a saída do Gemini se ela for JSON válido e contiver a mesma temperatura calculada; caso contrário (ou se o Gemini falhar — o nó está com *On Error: Continue*), usa o fallback.

**Como ativar:**
1. Gere uma chave em [Google AI Studio](https://aistudio.google.com/app/apikey).
2. No n8n, abra o nó Gemini → **Create New Credential** → *Google Gemini (PaLM) API* → cole a chave em **API Key** (host padrão `https://generativelanguage.googleapis.com`).
3. Se quiser, troque o modelo na lista (padrão: `gemini-2.5-flash`).
4. Clique com o botão direito no nó → **Activate** (ou selecione e pressione `D`) e salve.

## 🧪 Checklist

- [x] Testado com 3 cidades válidas
- [x] Testado com cidade inexistente (mensagem de erro)
- [x] Mensagem formatada com emoji, cidade e temperatura arredondada
- [x] Nenhum token ou chave no JSON ou no README

## 🛠️ Problemas comuns

- **`access to env vars denied`** → defina `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` e reinicie o n8n.
- **Bot não responde** → confira se `WEBHOOK_URL` é HTTPS público e se o workflow está **Active**.
- **Sempre cai no erro** → a chave da OpenWeather pode ainda não estar ativa (teste no navegador: `https://api.openweathermap.org/data/2.5/weather?q=london&appid=SUA_CHAVE`).
