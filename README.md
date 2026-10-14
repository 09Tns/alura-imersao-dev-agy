# 🧠 Imersão Dev Engenharia de Software na Era da IA: Agente de Estudos com Antigravity CLI + Google ADK

Roteiro do aulão das **Aulas 3 e 4** da **Imersão Dev Engenharia de Software na Era da IA**. Neste lab você vai usar o **Antigravity CLI (`agy`)**, a ferramenta de desenvolvimento com IA do Google no terminal, para construir um **agente especialista em estudos** com o **Google ADK** e o **agents-cli**.

O agente é **movido a skills**: o conhecimento dele mora em arquivos Markdown na pasta [`skills/`](skills/). Vamos usar a skill [`plano-de-estudos`](skills/plano-de-estudos/SKILL.md), que transforma um objetivo ("passar na entrevista de estágio em 3 semanas") em um **plano de estudos com revisão espaçada, checkpoints e um CSV que você importa direto no Google Agenda**.

---

## 🧩 Modelo mental

| Conceito | Equivalente no desenvolvimento tradicional | Papel no lab |
| :--- | :--- | :--- |
| **Antigravity CLI (`agy`)** | Par programador no terminal | Lê o projeto, gera código, roda comandos e revisa o que foi feito |
| **Google ADK** | Framework backend (FastAPI, Express) | Estrutura a lógica, as ferramentas e o ciclo de vida do agente |
| **agents-cli** | CLI de scaffolding (`create-react-app`, `rails new`) | Cria a estrutura do projeto e ensina o `agy` a programar com ADK (via skills) |
| **Skill (`SKILL.md`)** | Manual de procedimentos / regra de negócio | Conhecimento especializado que o agente carrega **sob demanda** |
| **`/grill-me`** | Refinamento de requisitos com o Tech Lead | Entrevista que alinha escopo, entradas e saídas **antes** de escrever código |
| **ADK Web** | Swagger / Postman com interface | Interface local para conversar com o agente e depurar cada chamada de ferramenta |

## 🗂️ Estrutura do repositório

```text
alura-imersao-dev/
├── README.md                    # este roteiro
├── AGENTS.md                    # contexto e regras que o agy lê automaticamente
└── skills/
    └── plano-de-estudos/        # ponto de partida do agente ADK
        ├── SKILL.md             # L1 (metadados) + L2 (gating e instruções)
        ├── references/
        │   ├── workflow.md      # L3: passo a passo para montar o plano
        │   └── tecnicas.md      # L3: técnicas de estudo baseadas em evidência
        └── assets/
            ├── plano_template.md  # L3: formato de saída + regras do CSV
            └── exemplo_plano.md   # L3: exemplo preenchido (few-shot)
```

**Como a skill funciona (divulgação progressiva):**

1. **L1, metadados:** o agente sempre vê só o `name` e a `description` da skill e decide se ela se aplica.
2. **L2, instruções:** ao ativar a skill, ele lê o corpo do `SKILL.md`, que traz a **regra de gating**. Sem *objetivo*, *prazo*, *disponibilidade* e *nível atual*, o agente só pergunta o que falta e não carrega mais nada.
3. **L3, recursos:** com tudo em mãos, ele carrega `workflow.md`, depois `tecnicas.md` e por fim o `plano_template.md`, sempre no momento certo e economizando tokens.

---

## ✅ Pré-requisitos

- **Navegador Google Chrome**.
- **Conta Google** com acesso a um **projeto do Google Cloud com faturamento ativo** (os créditos do [free trial](https://cloud.google.com/free) servem). É ele que dá acesso aos modelos usados pelo `agy`.
- **API Key do Google AI Studio**, usada pelo agente no ADK Web. Ela é criada na Etapa 6.
- **Conta no GitHub**, para a Etapa 9.

> ⚠️ **Custos:** o `agy` consome modelos pela Agent Platform do seu projeto. Acompanhe o faturamento e, se usar um projeto só para o lab, desative-o no final.

### Legenda dos ambientes

- 💻 **Terminal do Cloud Shell (`$`)**: comandos de shell.
- 🤖 **Prompt do agy (`agy >`)**: comandos e prompts dentro do Antigravity CLI.
- 🌐 **Navegador**: Console do Google Cloud, AI Studio, ADK Web e GitHub.

---

# 📘 AULA 3: Introdução ao Antigravity CLI

## Etapa 1: Preparar o Cloud Shell e habilitar as APIs

1. 🌐 Acesse o [Console do Google Cloud](https://console.cloud.google.com/), selecione o seu projeto no seletor do topo e clique em **Ativar o Cloud Shell** (ícone `>_` no canto superior direito). Na janela informativa, clique em **Continuar**.

2. 💻 Configure as variáveis de ambiente. Quando aparecer o pop-up **"Autorizar o Cloud Shell"**, clique em **Autorizar**:

    ```bash
    # Se o Cloud Shell não abriu com o projeto certo, rode antes:
    # gcloud config set project SEU_PROJECT_ID

    export PROJECT_ID=$(gcloud config get-value project)
    export REGION=us-central1
    gcloud config set compute/region $REGION

    cat << EOF >> ~/.bashrc
    export PROJECT_ID=${PROJECT_ID}
    export REGION=${REGION}
    export PATH="\$HOME/.local/bin:\$PATH"
    EOF

    echo "Projeto ativo: ${PROJECT_ID}"
    ```

3. 💻 **Habilite as APIs da Agent Platform** (Vertex AI), que dão acesso aos modelos Gemini usados pelo `agy`:

    ```bash
    gcloud services enable \
      aiplatform.googleapis.com \
      cloudaicompanion.googleapis.com \
      cloudresourcemanager.googleapis.com \
      serviceusage.googleapis.com
    ```

4. 💻 Confirme que as APIs estão ativas:

    ```bash
    gcloud services list --enabled --filter="name:(aiplatform.googleapis.com OR cloudaicompanion.googleapis.com)" --format="value(name)"
    ```

    A saída deve listar `aiplatform.googleapis.com` e `cloudaicompanion.googleapis.com`.

5. 💻 Clone este repositório:

    ```bash
    git clone https://github.com/johngarrido/alura-imersao-dev-agy.git ~/alura-imersao-dev
    cd ~/alura-imersao-dev
    ```

---

## Etapa 2: Instalar e autenticar o Antigravity CLI (`agy`)

> ℹ️ **Atalhos no terminal do Cloud Shell:** `CTRL+C` **interrompe** o programa em execução. Para abrir links, use `CTRL+Clique` (`CMD+Clique` no Mac). Para colar, use `CTRL+SHIFT+V` (`CMD+V` no Mac).

1. 💻 Instale o gerenciador de pacotes Python `uv`:

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    source $HOME/.local/bin/env
    ```

2. 💻 Instale o Antigravity CLI:

    ```bash
    curl -fsSL https://antigravity.google/cli/install.sh | bash
    export PATH="$HOME/.local/bin:$PATH"
    ```

3. 💻 Inicie o `agy` **na raiz do repositório**:

    ```bash
    cd ~/alura-imersao-dev
    agy
    ```

4. 🤖 Siga as etapas do setup interativo:

    | Passo | Tela | O que escolher |
    |---|---|---|
    | 1 | Inicialização | `2. Use a Google Cloud project` |
    | 2 | Autenticação | `1. Continue with Google Cloud` |
    | 3 | Permissões | Abra o link (`CTRL+Clique`), escolha sua conta, autorize e cole o código de volta no terminal |
    | 4 | ID do projeto | Cole o ID do seu projeto (é o valor de `echo $PROJECT_ID`) |
    | 5 | Select Google Cloud Location | `global` |
    | 6 | Select License | `1. Agent Platform` |
    | 7 | Color scheme | O tema que preferir (ex.: `dark`) |
    | 8 | Terms of Service & Data Use | `Done` |
    | 9 | Do you trust the contents of this project? | `Yes, I trust this folder` |

    Ao final você verá o prompt `agy >`.

> 🧯 **Erro `Permission 'aiplatform.endpoints.predict' denied` ou `Agent terminated due to error`?** Confira se a Etapa 1.3 rodou no projeto certo. Depois rode `agy auth logout` no terminal e abra o `agy` de novo.

---

## Etapa 3: Primeiros passos no `agy`

1. 🤖 Conheça os comandos disponíveis:

    ```text
    /help
    ```

    | Comando | O que faz |
    |---|---|
    | `/skills` | Lista as skills instaladas |
    | `/config` | Abre as configurações (ex.: permissão de ferramentas) |
    | `/grill-me` | Entrevista você para alinhar o design **antes** de codar |
    | `/clear` | Limpa o histórico da conversa |
    | `/exit` | Sai do `agy` |

2. 🤖 Teste a leitura de contexto. O `agy` lê o `AGENTS.md` automaticamente e consegue navegar pelo repositório:

    ```text
    Leia o arquivo skills/plano-de-estudos/SKILL.md e me explique em 3 tópicos como funciona a regra de gating desta skill e por que ela economiza tokens.
    ```

3. 🤖 Peça uma visão geral da skill inteira:

    ```text
    Quais arquivos da skill plano-de-estudos são carregados em cada nível (L1, L2 e L3)? Monte uma tabela.
    ```

4. 🤖 Saia do `agy`:

    ```text
    /exit
    ```

5. 🌐 Na barra do Cloud Shell, clique em **Abrir Editor** e navegue até `alura-imersao-dev/skills/plano-de-estudos/`. Observe o frontmatter YAML do `SKILL.md` (`name` e `description`) e os arquivos de `references/` e `assets/`. Depois clique em **Abrir Terminal** para voltar.

---

## Etapa 4: Instalar o agents-cli e habilitar as skills

O **agents-cli** traz as skills que ensinam o `agy` a criar, testar e publicar agentes ADK seguindo as boas práticas do Google.

1. 💻 Instale e configure o agents-cli:

    ```bash
    uvx google-agents-cli setup
    export PATH="$HOME/.local/bin:$PATH"
    ```

2. 💻 Abra o `agy` de novo:

    ```bash
    cd ~/alura-imersao-dev
    agy
    ```

3. 🤖 Confirme que as skills foram carregadas:

    ```text
    /skills
    ```

    Você deve ver **7 skills** `google-agents-cli-*` (`adk-code`, `scaffold`, `workflow`, `eval`, `deploy`, `observability` e `publish`). Pressione `ESC` para fechar.

4. 🤖 Libere o `agy` para criar arquivos e rodar comandos sem pedir confirmação a cada ação:

    ```text
    /config
    ```

    Em **Tool Permission**, selecione **`always-proceed`** e pressione `ESC`.

    > 💡 Em projetos reais, avalie com cuidado antes de usar `always-proceed`. No lab ele agiliza o fluxo.

---

## Etapa 5: Projetar e criar o agente com `/grill-me`

1. 🤖 Inicie a entrevista de design:

    ```text
    /grill-me
    ```

2. 🤖 Responda às perguntas do `agy` usando estas diretrizes. As perguntas podem vir em outra ordem ou com outras palavras:

    | Tema | Resposta sugerida |
    |---|---|
    | **Objetivo** | Um agente mentor de estudos que monta planos de estudo personalizados, com revisão espaçada, checkpoints de autoavaliação e uma agenda em CSV importável no Google Agenda. |
    | **Skills e ferramentas** | Usar a especificação de ADK Skills ([adk.dev/skills](https://adk.dev/skills/)) com `SkillToolset`, carregando a skill `skills/plano-de-estudos`. Sem ferramentas externas por enquanto. |
    | **Regra de gating** | O agente só carrega o workflow depois de ter objetivo, prazo, disponibilidade e nível atual. Se faltar algo, pergunta só o que falta. |
    | **Formato de saída** | Plano em Markdown seguindo `assets/plano_template.md`, com o bloco CSV no final. |
    | **Modelo e autenticação** | Modelo lido da variável `MODEL`. Execução local com API Key do Google AI Studio (`GEMINI_API_KEY`, `GOOGLE_GENAI_USE_VERTEXAI=FALSE`). Sem deploy. |
    | **Data** | A instrução do agente deve conter a data de hoje, para entender expressões como "segunda que vem". |

3. 🤖 Ao final do alinhamento, peça a criação do projeto:

    ```text
    Crie um projeto de agente ADK chamado agente-estudos, seguindo as convenções do agents-cli e a especificação de ADK Skills (adk.dev/skills). Copie a skill skills/plano-de-estudos para agente-estudos/app/skills/plano-de-estudos e carregue-a com load_skill_from_dir + SkillToolset. O modelo deve vir da variável de ambiente MODEL (padrão gemini-flash-latest). A instrução do agente deve incluir a data de hoje e reforçar a regra de gating da skill. Não crie configuração de deploy: vamos rodar só localmente com o ADK Web.
    ```

4. 🤖 Enquanto o `agy` trabalha, acompanhe o que ele está fazendo: quais skills do agents-cli ele consulta, quais comandos roda e quais arquivos cria. Quando terminar, peça um resumo:

    ```text
    Resuma a estrutura de pastas do agente-estudos e explique o que faz cada trecho do app/agent.py.
    ```

5. 🤖 Saia do `agy` com `/exit`.

> 🏁 **Fim da Aula 3:** você tem o `agy` configurado e um projeto ADK em `~/alura-imersao-dev/agente-estudos` com a skill de estudos integrada.

---

# 📗 AULA 4: Engenharia de software com IA e CLI

> 🔁 **Abriu uma nova sessão do Cloud Shell?** Os arquivos da sua home continuam lá. Rode `cd ~/alura-imersao-dev && source ~/.bashrc`.

## Etapa 6: Configurar a API Key do Google AI Studio

Para rodar no ADK Web, o agente vai falar com o Gemini direto pela **API do Google AI Studio**, usando uma API Key.

1. 🌐 Acesse [aistudio.google.com/apikey](https://aistudio.google.com/apikey), clique em **Create API key**, selecione o seu projeto do Google Cloud e copie a chave gerada.

2. 💻 Crie o `.env` do agente. O comando `read -s` pede a chave sem exibi-la na tela nem salvá-la no histórico do terminal:

    ```bash
    cd ~/alura-imersao-dev/agente-estudos

    read -s -p "Cole sua API Key do AI Studio e pressione ENTER: " GEMINI_API_KEY && echo

    cat << EOF > .env
    GOOGLE_GENAI_USE_VERTEXAI=FALSE
    GEMINI_API_KEY=${GEMINI_API_KEY}
    MODEL=gemini-flash-latest
    EOF

    unset GEMINI_API_KEY
    ```

    > 🔐 **Nunca** faça commit do `.env`. Ele já está no [`.gitignore`](.gitignore).

3. 💻 Instale as dependências do projeto:

    ```bash
    uv sync
    ```

4. 🌐 Clique em **Abrir Editor**, abra `agente-estudos/app/agent.py` e localize:
    - o `load_skill_from_dir(...)` apontando para `skills/plano-de-estudos`;
    - o `SkillToolset` na lista de `tools` do agente;
    - a leitura da variável `MODEL`;
    - a injeção da data de hoje na instrução.

    Volte para o terminal com **Abrir Terminal**.

---

## Etapa 7: Testar o agente no ADK Web

1. 💻 Suba o ADK Web liberando o proxy do Cloud Shell:

    ```bash
    cd ~/alura-imersao-dev/agente-estudos
    uv run adk web --allow_origins="*"
    ```

    > 💡 Alternativa: `agents-cli playground`.

2. 🌐 Quando aparecer `ADK Web Server started ... http://127.0.0.1:8000`, dê `CTRL+Clique` no link. Outra opção: **Visualização na Web → Alterar porta → `8000`**. No seletor do canto superior esquerdo, escolha o agente `app`.

3. 🌐 Rode os três cenários abaixo. Em cada um, abra a aba **Events / Trace** e observe **quais ferramentas** o agente chamou (`load_skill`, `load_skill_resource`) e **em que momento**.

    **Cenário 1: Gating (informação incompleta)**

    ```text
    Quero organizar meus estudos de JavaScript.
    ```

    ✅ **Esperado:** o agente pergunta prazo, disponibilidade e nível, e **não** chama `load_skill_resource`.

    **Cenário 2: Plano completo**

    ```text
    Quero aprender JavaScript para construir meu primeiro projeto de portfólio em 5 semanas. Começo segunda que vem. Posso estudar segunda, quarta e sexta, 2h por dia às 19h. Já sei HTML e CSS, mas nunca programei.
    ```

    ✅ **Esperado:** o agente carrega `workflow.md`, `tecnicas.md` e `plano_template.md`, e entrega o plano com módulos, cronograma, checkpoints e o CSV.

    **Cenário 3: Replanejamento**

    ```text
    Perdi as sessões de quarta e sexta da primeira semana. Ajusta o plano?
    ```

    ✅ **Esperado:** o agente usa os dias de margem, empurra o conteúdo sem empilhar tudo num dia só e devolve a tabela e o CSV atualizados.

4. 🌐 **Efeito prático:** copie o bloco CSV, salve como `plano-de-estudos.csv` e importe no [Google Agenda](https://calendar.google.com/) em **Configurações → Importar e exportar → Importar**. Suas sessões de estudo aparecem na agenda! 🎉

5. 💻 Deixe o ADK Web rodando nesta aba. Vamos usá-lo na próxima etapa.

---

## Etapa 8: Evoluir o agente com o `agy`

Nesta etapa você usa o `agy` como engenheiro de software: revisar código, achar problemas, escrever testes e evoluir funcionalidades.

1. 💻 Abra uma **segunda aba** no Cloud Shell (ícone `+`) e inicie o `agy` na raiz do repositório:

    ```bash
    cd ~/alura-imersao-dev
    agy
    ```

2. 🤖 **Revisão de código:**

    ```text
    Faça uma revisão de código do agente-estudos. Verifique se ele segue as diretrizes do AGENTS.md e as boas práticas de ADK Skills. Liste os problemas por severidade (alta, média, baixa) antes de corrigir qualquer coisa.
    ```

3. 🤖 **Identificar problemas a partir de um comportamento real.** Escolha algo que não ficou bom nos testes da Etapa 7 (um CSV com datas erradas, uma pergunta repetida, uma sessão sem entregável) e descreva para o `agy`:

    ```text
    No teste do ADK Web, o agente [descreva o problema]. Investigue a causa: é a instrução do agente, o SKILL.md ou algum arquivo de references/assets? Proponha a correção e aplique.
    ```

    > 💡 Se a correção for na skill, lembre que o agente usa a cópia em `agente-estudos/app/skills/`. Peça ao `agy` para manter as duas cópias sincronizadas.

4. 🤖 **Testes automatizados:**

    ```text
    Crie testes com pytest para o agente-estudos que validem: (1) a skill plano-de-estudos carrega sem erros com load_skill_from_dir; (2) o nome da pasta bate com o name do frontmatter; (3) todos os arquivos citados no SKILL.md e no workflow.md existem; (4) o root_agent tem o SkillToolset configurado. Rode os testes e corrija o que falhar.
    ```

    > Para ir além, peça ao `agy` para usar a skill `google-agents-cli-eval` e criar casos de avaliação para os cenários 1 e 2 da Etapa 7.

5. 🤖 **Nova funcionalidade: uma segunda skill.** Peça ao `agy` para criar uma skill complementar e observe o agente escolher entre as duas pela `description`:

    ```text
    Crie uma nova skill ADK chamada revisao-ativa em skills/revisao-ativa (e a cópia em agente-estudos/app/skills/), seguindo o mesmo padrão da plano-de-estudos: gating (só age quando a pessoa colar o material de estudo), references/workflow.md e assets/template.md. Ela deve gerar flashcards no formato CSV de importação do Anki e um quiz de 5 perguntas com gabarito comentado. Registre a skill no SkillToolset do agente.
    ```

6. 🌐 Volte à primeira aba, reinicie o ADK Web (`CTRL+C` e `uv run adk web --allow_origins="*"`) e teste:

    ```text
    Cola aqui: "Closures em JavaScript são funções que lembram o escopo onde foram criadas, mesmo depois que esse escopo terminou de executar..." Me ajuda a revisar isso?
    ```

---

## Etapa 9: Versionar no GitHub

1. 💻 Verifique o GitHub CLI e faça login (escolha **GitHub.com → HTTPS → Login with a web browser**):

    ```bash
    gh --version
    gh auth login
    ```

2. 🤖 No `agy`, peça para versionar o projeto:

    ```text
    Prepare o repositório para o GitHub: confira se o .env e o .venv estão no .gitignore e se não há nenhuma API Key em arquivos versionados. Depois crie commits pequenos e com mensagens descritivas (estrutura do agente, testes, skill revisao-ativa), crie um repositório público chamado agente-estudos-imersao na minha conta com o gh, aponte o remote origin para ele e faça o push.
    ```

    <details>
    <summary>Prefere fazer na mão? Clique aqui.</summary>

    ```bash
    cd ~/alura-imersao-dev
    git status                      # confira: nenhum .env na lista!
    git add .
    git commit -m "feat: agente de estudos com ADK e skill plano-de-estudos"
    git remote remove origin
    gh repo create agente-estudos-imersao --public --source=. --remote=origin --push
    ```

    </details>

3. 🌐 Abra o repositório no GitHub e confirme que **não há** nenhum `.env` publicado.

> 🏁 **Fim da Aula 4:** você construiu, testou, revisou, evoluiu e versionou um agente ADK sem sair do terminal, com o `agy` como par programador.

---

## 🧯 Solução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| `agy: command not found` | `PATH` desatualizado | `export PATH="$HOME/.local/bin:$PATH"` ou `source ~/.bashrc` |
| `Permission 'aiplatform.endpoints.predict' denied` | API não habilitada, projeto errado ou login com outra conta | Refaça a Etapa 1.3, rode `agy auth logout` e entre de novo |
| `/skills` não mostra as `google-agents-cli-*` | `agents-cli setup` não concluiu | Rode `uvx google-agents-cli setup` de novo e reabra o `agy` |
| ADK Web abre página em branco | Cookies de terceiros bloqueados | Permita cookies para `cloudshell.dev` no ícone de olho da barra de endereços |
| ADK Web não lista o agente | `adk web` rodando na pasta errada | Rode dentro de `~/alura-imersao-dev/agente-estudos` |
| Erro ao carregar a skill | Nome da pasta diferente do `name` no frontmatter | A pasta tem que se chamar exatamente `plano-de-estudos` |
| `API key not valid` / `401` | `.env` com chave errada ou sem `GOOGLE_GENAI_USE_VERTEXAI=FALSE` | Refaça a Etapa 6.2 e reinicie o ADK Web |
| `429 RESOURCE_EXHAUSTED` | Limite do nível gratuito do AI Studio | Aguarde 1 minuto e tente de novo, ou ative o faturamento na chave |
| Datas erradas no CSV | O modelo não sabe a data atual | Confira se a instrução do agente injeta a data de hoje (Etapa 5.3) |

---

## 📚 Referências

- [Antigravity CLI](https://antigravity.google/)
- [Google ADK: documentação](https://adk.dev/)
- [ADK Skills](https://adk.dev/skills/) · [Especificação Agent Skills](https://agentskills.io/specification)
- [Google AI Studio: API Keys](https://aistudio.google.com/apikey)
