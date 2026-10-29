# Servidor de WhatsApp — Kairós (v5)

Este é o "motor" que mantém o WhatsApp da empresa conectado 24 horas por dia.
Ele recebe as mensagens dos clientes, responde pela IA, envia as respostas dos
atendentes e dispara as cobranças e confirmações de pagamento.

> ⚠️ **Ele precisa ficar ligado o tempo todo e precisa de um disco próprio.**
> Hospedagens que "dormem" por inatividade ou que apagam o disco a cada deploy
> (Vercel, Netlify, Cloudflare, funções serverless) **não servem** — a sessão do
> WhatsApp morre e você precisa escanear o QR Code de novo toda hora.

---

## 1. Qual hospedagem usar

| Plataforma | Custo mensal | Dorme? | Disco | Veredito |
|---|---|---|---|---|
| **Render (plano Starter)** | ~US$ 7,25 (≈ R$ 39) | Não | Sim (US$ 0,25/GB) | **Padrão do projeto.** Só cliques, sem terminal, sem arquivo de porta para editar |
| Railway | US$ 5 (≈ R$ 27) | Não | Sim | Funciona igual e é um pouco mais barato; alternativa boa |
| VPS (Hetzner/Contabo) | €4–5 (≈ R$ 25–30) | Não | Sim (disco inteiro) | Mais barato no longo prazo, mas exige terminal |
| Oracle Cloud Free | R$ 0 | Não | Sim | Existe plano gratuito, mas o cadastro é chato e a máquina é lenta |
| Fly.io | ~US$ 3,84 (≈ R$ 21) | Não | Sim | Foi a origem dos erros de porta/IP. **Não recomendado** |

**Escolha do projeto: Render, plano Starter.** Cumpre os três requisitos de
produção (processo sempre vivo, disco persistente, reinício automático) e é o
mais difícil de errar: o `render.yaml` já vem pronto, não existe tela de
`internal_port` para editar e o endereço público já nasce funcionando.

Sobre "conexão boa": o que pesa aqui não é o país do servidor, e sim ele nunca
dormir, ter disco e ficar perto do Backend. Use a região **virginia** (EUA, costa
leste): é a mais próxima do Brasil (~120 ms). A conversa com o WhatsApp é feita
pelos servidores do WhatsApp, não pelo Brasil.

Custos que podem aparecer além da mensalidade:
- Disco de 1 GB: US$ 0,25. A sessão ocupa poucos MB, 1 GB sobra.
- Tráfego de saída: o volume deste serviço é ínfimo.

---

## 2. Publicar no Render (caminho padrão)

Tudo pelo navegador, sem terminal. Leva ~10 minutos, uma vez só.

1. **GitHub.** Suba esta pasta em um repositório novo (pode ser privado).
   `package.json`, `Dockerfile` e `render.yaml` precisam ficar na **raiz** do
   repositório, não dentro de uma subpasta.
2. **Criar o serviço.** render.com → **New** → **Blueprint** → conecte o GitHub
   → escolha o repositório. O Render lê o `render.yaml` e já monta o serviço com
   imagem Docker, plano, disco e checagem de saúde.
3. **Revisar.** Confirme na tela de revisão:
   - plano **Starter** (o Free **não** serve)
   - disco montado em **`/var/data`**, 1 GB
   - região **virginia**
4. **Variáveis.** O Render pede os valores das que estão marcadas como secretas:

   ```
   WPP_API_KEY=<sua chave>
   WPP_AGENT_SECRET=<a mesma chave da função wpp-agente>
   SUPABASE_URL=https://SEU-BACKEND
   SUPABASE_ANON_KEY=<chave pública>
   SUPABASE_SERVICE_KEY=<chave de serviço, recomendada>
   ```

   Já vêm prontos no blueprint: `WPP_AUTH_DIR=/var/data/wpp-auth` e
   `TZ=America/Sao_Paulo`. **Não cadastre `PORT`** — o Render injeta sozinho e o
   servidor lê `process.env.PORT`.
5. **Deploy.** O primeiro build leva de 3 a 8 minutos (instala o Baileys e
   compila dependências). Espere o status virar **Live**.
6. **Endereço.** Aparece no topo do serviço, no formato
   `https://seu-servico.onrender.com`.
7. **Confira.** Abra `https://seu-servico.onrender.com/api/health`. Precisa
   aparecer `"ok":true` e `"db":"ok"`. Depois cole só o endereço na aba
   **Conexão** do painel.

Se o build falhar, abra **Logs**: 9 em cada 10 vezes é uma variável faltando ou
o disco não montado em `/var/data`.

> Por que o plano Starter e não o Free: no Free o serviço hiberna após 15 min
> sem acesso e não tem disco. Hibernou = sessão do WhatsApp caiu = QR Code de
> novo. Não existe contorno; é limitação do plano.

---

## 2.1 Alternativa: Railway (também funciona bem)

1. railway.app → **New Project** → **Deploy from GitHub repo** → escolha o
   repositório. Ele encontra o `Dockerfile` sozinho.
2. **Settings → Region:** `us-east4 (Virginia)`.
3. **Settings → Volumes → Add Volume:** montagem em **`/data`**.
4. **Variables:** as mesmas cinco + `WPP_AUTH_DIR=/data/wpp-auth`.
   Não cadastre `PORT`.
5. **Settings → Networking → Generate Domain.**
6. Abra `/api/health` e confirme `"ok":true`.

Diferença única: no Railway o disco é `/data`; no Render é `/var/data`. Por isso
o `WPP_AUTH_DIR` muda entre as duas.

---

## 3. Publicar em VPS próprio (mais barato no longo prazo)

```bash
# na VPS, com Docker instalado
docker compose up -d --build
```

O `docker-compose.yml` já cria o volume e reinicia o serviço sozinho.
Para o HTTPS, coloque um proxy (Caddy ou Nginx) na frente apontando para a porta 8080.

---

## 4. Variáveis de ambiente

| Variável | Obrigatória | O que é |
|---|---|---|
| `PORT` | não | Porta HTTP (padrão 8080). O Render/Railway injetam sozinhos. O servidor também escuta em 3000 e 3001 por segurança. |
| `WPP_API_KEY` | **sim** | Senha que o painel usa para falar com este servidor. |
| `WPP_AGENT_SECRET` | **sim** | A mesma chave configurada na função `wpp-agente` do Backend. |
| `SUPABASE_URL` | **sim** | Endereço do Backend (Readdy). |
| `SUPABASE_ANON_KEY` | **sim** | Chave pública do Backend. |
| `SUPABASE_SERVICE_KEY` | recomendada | Chave de serviço do Backend. Sem ela o servidor usa a pública e perde acesso a partes do banco. |
| `WPP_AUTH_DIR` | **sim** | Pasta da sessão. **Precisa estar dentro do disco/volume.** Render: `/var/data/wpp-auth` · Railway/VPS: `/data/wpp-auth` |
| `WPP_PUBLIC_URL` | não | URL pública do serviço. É detectada automaticamente no Render/Railway. |
| `TZ` | não | `America/Sao_Paulo` para os horários saírem certos. |

Nunca escreva essas chaves em arquivos versionados. Use os "Secrets" da plataforma.

---

## 5. Como saber se está tudo bem

- `SUA_URL/api/health` → deve responder `{"ok":true,...,"db":"ok"}`.
- Na aba **Conexão** do painel, o diagnóstico mostra versão, tempo no ar,
  sessão salva, contagem de mensagens e a data da última cobrança.
- Configure um monitor gratuito (UptimeRobot, BetterStack) apontando para
  `/api/health` a cada 5 minutos, com alerta por e-mail.

O que o servidor já garante sozinho:
- **Sem mensagem duplicada**: cada parcela é "reservada" no banco antes do envio.
- **Cobrança não se perde**: se o serviço estiver fora do ar às 11h, ele envia
  assim que voltar (e só uma vez por dia, mesmo reiniciando várias vezes).
- **Reconexão automática** com espera progressiva.
- **Fila do atendente humano** entregue em até 5 segundos.
- **Desligamento limpo**: em deploy/reinício ele fecha a sessão antes de morrer,
  evitando arquivo de credencial corrompido.

---

## 6. Backup e volta atrás

**Backup da sessão (opcional, mas útil):**
No Render, o disco pode ser baixado em **Disks → seu disco**. No Railway, em
**Settings → Volumes → Download**. No VPS, basta copiar a pasta `WPP_AUTH_DIR`.
Se perder a sessão, basta escanear o QR Code novamente — nada de dados de
clientes é perdido, porque tudo fica no Backend, não neste servidor.

**Voltar para a hospedagem anterior:** este servidor não depende de nenhuma
plataforma específica. Publique esta mesma pasta no host antigo, aponte o disco
para a pasta da sessão e cole a URL nova na aba **Conexão**. Nada muda no código.

**Nunca apague o serviço antigo antes de o novo responder** em `/api/health`
e de o botão **Enviar teste** chegar no seu celular.

---

## 7. Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| Build falha no Render | Faltou variável ou `Dockerfile` não está na raiz | Veja **Logs**; confira a raiz do repositório |
| "Live" mas a URL não responde | Disco não montado em `/var/data` | Confira em **Disks**; o `WPP_AUTH_DIR` precisa apontar para dentro dele |
| Conecta e cai em minutos | Sem disco (sessão não persiste) | Crie o disco e ajuste `WPP_AUTH_DIR` |
| Serviço "dormiu" | Plano Free | Mude para **Starter** |
| QR não aparece | Serviço ainda subindo | Espere o deploy ficar Live e clique em Conectar |
| Conectado mas não responde | Falta a chave da IA no Backend | Cadastre `OPENAI_API_KEY` no Backend |
| Cobrança não saiu hoje | Fora do horário ou sem conexão | Veja `last_cobranca` no diagnóstico |

---

## 8. Portas (só se algum dia der "Request failed")

O servidor escuta em **8080, 3000 e 3001** ao mesmo tempo, e usa
`process.env.PORT` quando a plataforma injeta uma porta. Isso cobre Render,
Railway, Fly e VPS sem nenhuma configuração. Se algum dia a URL ficar muda com o
serviço ligado, é sinal de que a hospedagem está batendo numa porta onde ninguém
escuta — nesse caso, confira se alguém cadastrou `PORT` manualmente com um valor
errado e remova essa variável.