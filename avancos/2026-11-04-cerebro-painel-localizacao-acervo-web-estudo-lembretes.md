# 04/11/2026 — Cérebro (M1/M2), painel (devices, localização, cérebro), acervo de documentos, pesquisa na web, estudo no cérebro e lembretes que avisam

Projeto: `iobraza` · branch `feat/cli-unificado` · modelo `qwen35-27b-abl` (vLLM, 3090) · 17 commits no dia (`3241ee3` → `507c36f`).

## Resumo do dia

| # | Avanço | Commit(s) | Validado no modelo real? |
|---|--------|-----------|--------------------------|
| 1 | Cérebro M1: entidades, observações, relações e busca | `3241ee3` | sim |
| 2 | Cérebro M2: registros (o que aconteceu, com data e números) | `e1fc550` | sim (roteiro academia) |
| 3 | Painel P1: `/painel` servido pelo agente, página Devices | `ffc6668` | sim (uso) |
| 4 | Painel P2: permissões, `get_location`, histórico, mapa | `38af8a4`, `c376d0b`, `d739156`, `9ea5f49` | sim (S24 + Google Timeline) |
| 5 | Painel P3: aba Cérebro (grafo, ficha, linha do tempo, revisar) | `4b9f821`, `0264191`, `e05246f` | sim (uso) |
| 6 | Fix do resumidor (o agente inventava) | `5763161` | sim |
| 7 | Acervo de documentos + `search_docs`/`read_doc` | `cb3b59d`, `9654cee` | sim (Scrum, XP, COCOMO) |
| 8 | Envio de documentos (CLI, terminal, painel) | `a6b1e55` | sim (regulamento da academia) |
| 9 | Pesquisa na web: SearXNG próprio + `web_search`/`web_fetch` | `7f740ae` | sim (Scrum Guide 2017×2020) |
| 10 | Tutor grava o estudo no cérebro | `2be5df9` | sim |
| 11 | Lembretes que avisam (`remind` + agendador) | `507c36f` | sim (aviso chegou às 15:34:05) |
| 12 | Mapa: presets preenchem De/Até, horário em cada ponto | `1919599` | parcial (só o código; tela não vista) |
| 13 | Tutor sempre raciocina + CLI mostra o raciocínio colapsado | `507c36f` | sim (A/B) |

---

## 1. Cérebro M1: o que o agente sabe do dono

**Escolha:** tudo num SQLite só, `data/cerebro.db`, com estas tabelas:
- `tipos`: normalizados, com área;
- `entidades`: uma linha por nome;
- `observacoes`: nunca são apagadas; corrigir = invalidar a antiga e criar a nova com `substitui`;
- `relacoes`;
- `obs_fts`: busca sem acento.

As tools são `remember` (com `substitui` e aviso de "já existia parecido") e `recall` (mostra o "antes:" das correções). Em todo turno o agente recebe o **núcleo** (observações sobre "Você", até ~1600 chars) mais **6 observações relevantes** para a fala atual.

**Caminhos até chegar nela:**
- *Memória antiga (facts):* falhava com dados reais. Havia 22 facts com categorias inventadas, e nenhuma era `profile`, então o perfil chegava **vazio**. O `UNIQUE(category,key)` sobrescrevia: o histórico do supino se perdia. A busca usava AND da frase inteira e quase nunca casava.
- *Vault Markdown estilo Obsidian/Basic Memory (proposta 1):* o MD seria a fonte e o SQLite o índice. **Descartado pelo dono:** sem Markdown como fonte, só export para backup e leitura.
- *Cérebro só em SQLite com painel próprio (proposta revisada):* **escolhido**. Os tipos são abertos mas normalizados, porque a memória é geral (academia, estudos, trabalho, finanças), não uma lista fixa.

**Observações:**
- Bug latente achado: a memória ia numa **2ª mensagem system** e o Qwen dava 400. A memória vinha vazia sem ninguém perceber. Agora vai dentro da 1ª system.
- O aviso "já existia parecido" funciona: o modelo corrigiu com `substitui` já na 1ª tentativa.

**Pontos de atenção:**
- `scripts/medir_toolcall.py` quebrou com a remoção de `memory_service`. É protótipo e não deve ser editado.
- O núcleo vai em todo turno: precisa continuar pequeno.

**Melhorias:**
- Busca semântica (embeddings), além do FTS.
- M4: conversas viram um diário resumido.

## 2. Cérebro M2: registros (treino, série, estudo)

**Escolha:** tabela `registros`. Uma *sessão* (o treino do dia) é um registro com `eh_sessao=1` e `fim` nulo enquanto dura; as séries apontam para ela. Tools `log` (abrir, registrar, fechar, corrigir) e `query_log` (último, período). Datas faladas ("ontem 18:40", "28/10", "semana"). O bloco de memória mostra "Agora:" e "Em andamento:".

**Caminhos comparados:**
- Treino como *fact* (o que existia): sobrescrevia, sem histórico.
- Registro com sessão: **escolhido**. Dá linha do tempo e "último treino de peito" com os números exatos.

**Observação (lição medida):**
- Escrever no prompt "os registros entram na sessão **sozinhos**" fez o modelo **não chamar `log`** e responder "Anotado" sem gravar nada.
- A redação certa: **cada evento exige a sua chamada**; a sessão só agrupa.

**Pontos de atenção:**
- Sessão esquecida aberta: o sistema considera "em andamento" só por algumas horas.
- O modelo precisa usar sempre o mesmo nome para o mesmo exercício.

**Melhorias:** sugestão de nome canônico ("Supino reto" × "supino") na hora de registrar.

## 3. Painel P1: devices

**Escolha:**
- Painel servido pelo próprio agente em `/painel`, em HTML/JS puro, sem build.
- Acesso por cookie gerado a partir da chave, **sem a chave dentro do cookie**.
- A página Devices mostra conectado, último sinal e ferramentas, e notifica usando o mesmo `notify` do agente.

**Caminho comparado:** app separado com build (React etc.) contra páginas estáticas servidas pelo FastAPI. Escolhemos as páginas estáticas: zero infraestrutura nova, e o deploy é copiar arquivos.

**Melhoria:** recarregar sozinho ao trocar de versão. Hoje precisa de Ctrl+F5.

## 4. Painel P2: permissões e localização

**Escolha:**
- As permissões são pedidas **uma vez no setup** (`python -m companion permissoes`) e enviadas no handshake.
- Tool `get_location` com fontes em cascata: Termux GPS → rede → última conhecida; Windows GeoCoordinateWatcher; Linux GeoClue; posição fixa para device parado.
- No Termux, coleta a cada 15 min com **fila local + ack**: nada se perde se a rede cair.
- Histórico no `devices.db`. Mapa Leaflet vendorizado, trilha por intervalo, importador da Linha do tempo do Google (3 formatos).

**Caminhos comparados:**
- Pedir permissão a cada uso contra uma vez no setup: escolhemos uma vez. No celular, um pedido no meio da conversa trava o fluxo.
- Enviar a posição na hora contra fila com ack: escolhemos a fila. O Tailscale do celular cai, e o envio direto perdia pontos.

**Observações / gotchas:**
- No Windows, um processo filho de console aberto pelo companion escondido morria (`0xC0000142`). Resolvido com `CREATE_NO_WINDOW`.
- O GeoClue em pt_BR devolve vírgula decimal. Quando ele cai na localização por IP, o ponto é marcado `geoclue-ip`.
- A "linha limpa" do mapa remove visitas, saltos e tremido.

**Pontos de atenção:** atualizar o S24 (pull + setup) a cada mudança do companion.

**Melhorias (já feitas hoje à tarde):** presets preenchem De/Até; horário em cada ponto (item 12).

## 5. Painel P3: aba Cérebro

**Escolha:**
- Grafo force-graph vendorizado: "Você" no centro, hubs de área, entidades e partículas nas ligações.
- Ficha editável com histórico: corrigir = substituir; também invalidar e apagar de vez.
- Linha do tempo com gráfico por assunto e tela Revisar.
- Depois evoluiu para **lobos/neurônios** com 3 níveis de expansão (Áreas → Grupos → Tudo), uma barra de filtros única (Área, Tipo, Período) para todas as visões, uma busca que acende regiões e a visão em Lista.

**Caminhos comparados:**
- Grafo "plano" com todas as entidades: ilegível com dados reais.
- Expansão por níveis (lobo → grupo → item): **escolhida** depois das iterações de feedback.
- Filtros por visão contra barra única: escolhemos a barra única. Antes os indicadores não acompanhavam a busca.

**Observações / gotchas:**
- Classe com `display:grid` anula o `[hidden]` → regra global `[hidden]{display:none!important}`.
- Entidade criada como alvo de ligação nasce "geral" → `_promover` dá o tipo quando ele aparece.

**Ferramenta de teste:** `ops/cerebro_exemplo.py status|popular|limpar` (com `fonte='exemplo'`) enche o cérebro para testar a interface **sem tocar no dado real**. O cérebro real ficou só com "Você" e o treino de peito real.

**Melhoria:** botão "Revisar" com contagem do que mudou desde a última visita.

## 6. Fix do resumidor: o agente inventava conteúdo

**Problema:** `ContextManager.summarize` resumia **toda** saída de ferramenta acima de 256 tokens, com o raciocínio **ligado**. O "resumo" que voltava era o raciocínio cortado do modelo ("We need answer…"), e o agente inventava a partir desse lixo.

**Escolha:**
- Resumir só acima de ~2500 tokens, sempre sem raciocínio.
- Uma lista `_NAO_RESUMIR` de ferramentas cujo texto tem que chegar íntegro: `search_docs`, `read_doc`, `recall`, `query_log`, `get_location`, `list_devices`, `web_search`, `web_fetch`.

**Caminhos comparados:** só subir o limite (ainda corrompia saídas grandes) contra limite + `no_think` + exceções. Escolhemos a combinação.

**Ponto de atenção:** ferramenta nova cujo texto precisa ser fiel tem que entrar em `_NAO_RESUMIR`.

## 7. Acervo de documentos (material real do tutor)

**Escolha:**
- Acervo genérico em Markdown (`data/acervo/<coleção>/`), indexado em `data/acervo.db`.
- Estrutura: documentos → seções → partes de até 3000 chars, cortadas em parágrafos.
- Busca FTS5 BM25 sem acento, devolvendo uma entrada por seção.
- Tools `search_docs` e `read_doc` (sumário, seção, parte).
- O conteúdo da Estácio é "desembrulhado": acordeões e cards viram negrito, créditos são removidos e as questões ficam em ordem.
- Regra no tutor: **material de disciplina vem do acervo, com fonte citada**.

**Caminhos comparados:**
- Material inteiro no prompt: não cabe (o tema tem 60–75 KB) e o modelo se perde.
- RAG por embeddings: mais infraestrutura; para termos técnicos exatos (nomes de pilares, siglas) o BM25 lexical acerta melhor.
- Índice por seções com leitura em duas etapas (buscar → ler a seção): **escolhido**. O modelo navega como um humano e cita a fonte.

**Medido no modelo real:** 3 pilares, timeboxes, 7 papéis do XP e a tabela do COCOMO saíram fiéis ao material, com fonte.

**Pontos de atenção:**
- Detalhes do HTML da Estácio quebram a limpeza: NBSP, enunciado como heading, H1/H2 soltos dentro de módulo. Cada caso exigiu um ajuste.
- Ferramenta de manutenção: `ops/acervo.py importar-estacio|importar|reindexar|status|buscar`.

**Melhorias:** busca híbrida (BM25 + embeddings) para perguntas parafraseadas.

## 8. Enviar documentos (CLI, terminal, painel)

**Escolha:**
- Três portas para a mesma rota `POST /acervo`: `/documento` no CLI, `companion acervo add` no terminal e a aba Acervo no painel (arrastar e soltar).
- O **markitdown** converte PDF, Word, PPT, HTML e planilhas.
- O agente recebe em todo turno a lista dos documentos enviados (fora Estácio e Web) e consulta quando o assunto aparece, mesmo sem o documento ser citado.

**Medido:** pergunta indireta sobre o regulamento da academia respondida a partir do documento.

**Observação:** o campo **Coleção** é a pasta de destino do próximo envio, não renomeia nada. Um nome novo cria um card novo.

**Melhorias:**
- Renomear coleção pelo painel.
- Trocar o campo de texto por uma lista das coleções existentes + "nova…".

## 9. Pesquisa na web: SearXNG próprio

**Escolha:**
- **SearXNG em Docker** (`searxng`, `127.0.0.1:8888`, `--restart unless-stopped`), configurado em `ops/searxng/settings.yml`, com o segredo fora do repo.
- A estabilidade é garantida no `braza start`, no boot e no watchdog: `EnsureDocker` sobe o Docker Desktop e espera até 180 s, e `EnsureSearx` cria ou inicia o container. Flag `--busca`. Linha "Busca:" no status.
- `web_search` busca e remove duplicados.
- `web_fetch` extrai o conteúdo principal com **trafilatura** (sem menu e rodapé), guarda no acervo (coleção `web`) e, em página longa com `pergunta`, **devolve só os 3 trechos mais relevantes** (BM25 por parte, texto original).

**Caminhos comparados:**
- *SearXNG nativo no WSL:* começou, mas foi interrompido. Como o dono já usa Docker no Windows, escolhemos Docker, garantido junto com as outras proteções de startup.
- *Abordagens de pesquisa:*
  - leitor que resume a página com LLM: perde a fidelidade;
  - deep research com subagentes paralelos: caro para uma GPU;
  - **reranking de trechos estilo Perplexity: escolhido**, reaproveitando o BM25 do acervo.
- *Leitura sequencial (antes):* página longa = 6 `read_doc` em sequência. *Com `pergunta`:* 2 `web_fetch` + 1 `search_docs` no mesmo teste.
- *User-Agent:* fingir ser navegador para passar do 403 da Wikipédia foi **recusado** (contornar bloqueio de robô). Mantivemos um UA honesto e configurável (`web_user_agent`); 401/403/429 voltam como "recusou robôs, use outro resultado".

**Observações / gotchas:**
- O credential helper do Docker Desktop falha via SSH ("logon session does not exist"). O `pull` precisa rodar na sessão do usuário (tarefa `schtasks /it` de uma vez).
- O primeiro reinício do watchdog falhou por causa das aspas no `.vbs` inline e deixou o watchdog **inativo**. Resolvido copiando um `.vbs` e rodando via schtasks. Recuperação testada: container parado voltou em 6 s.
- No server, o `python-dateutil` estava registrado mas sem os arquivos: reinstalado com `uv pip install --reinstall`.

**Pontos de atenção:**
- Mesmo com a regra, o modelo **citou fontes que não abriu** (só vieram na lista da busca) e preencheu linhas da comparação de cabeça ("Sprint Review obrigatória em 2020"). Entrou uma regra explícita, **ainda não re-medida**: citar só o que abriu e não completar tabela de cabeça.
- Container parado e `settings.yml` ausente quebram a busca. O `braza restart` completo resolve.

**Melhorias:**
- Re-medir a fidelidade da comparação (2–3 perguntas atuais).
- Guard determinístico: link citado na resposta tem que ter passado por `web_fetch` no turno.

## 10. Tutor grava o estudo no cérebro

**Escolha:**
- **Abrir e fechar a sessão de estudo:** o próprio modelo faz, com `log(abrir, estudo, disciplina, tema)` e `log(fechar)` no "por hoje é só". Nos testes ele chamou sozinho.
- **Acertos e dificuldades:** o **servidor** registra. Depois de cada turno do tutor, em segundo plano, uma chamada curta (sem raciocínio, JSON) lê a correção e extrai `{tema, acertos, total, dificuldades}`. Grava `exercicios · "acertou 1 de 3"` e cada dificuldade como observação do tema, ligada à disciplina. Se nenhuma sessão estiver aberta, abre uma.
- Se o modelo já chamou `log(exercicios)` no turno, não duplica.

**Caminhos comparados (todos medidos no modelo real):**
1. Regra no bloco do tutor ("corrigiu → log + remember"): **ignorada**. Corrigiu bem e não registrou.
2. Mesma regra colada na regra de correção ("AO CORRIGIR, na mesma vez…"): **ignorada de novo**. O raciocínio mostra a atenção toda no aluno.
3. Marcador escondido no texto (`<!--acertos=…-->`, como o `calc=` do gabarito): descartado. No chat o CLI mostraria o marcador.
4. **Extração pelo servidor depois do turno: escolhido.** Determinístico no gatilho, com o LLM só para ler a correção. Resultado: "acertou 1 de 3" + 3 dificuldades corretas ("acha que o Scrum Master cobra a equipe na daily").

**Observações:**
- "Meio certo conta como erro". O texto do registro guarda as dificuldades para a linha do tempo.
- Dados de teste removidos do cérebro real (registros 644–646, observações 115–117, entidades 128–129).

**Pontos de atenção:**
- A extração roda em **todo** turno do tutor: uma chamada curta extra na mesma GPU (~1–3 s, em segundo plano).
- Se o tutor corrigir de forma ambígua, a contagem pode sair errada. O prompt pede para deixar claro, questão por questão, o que acertou e o que errou.

**Melhorias:**
- Disparar a extração só quando a mensagem anterior do tutor tinha perguntas, economizando GPU.
- Usar as dificuldades gravadas para a próxima sessão começar revisando o ponto fraco.

## 11. Lembretes que avisam

**Escolha:**
- Tabela `lembretes` no `cerebro.db`: texto, `proximo` (vencimento), repetição, antecedência, dia original, `avisado_para`.
- Tool `remind` (criar, listar, cancelar).
- **Agendador no lifespan**, a cada 30 s: manda `notify` para os devices configurados (`lembrete_devices`) ou, sem configuração, para todos os conectados que sabem notificar.
- Interpretador de datas futuras: "dia 10", "amanhã 9h", "sexta 18:00", "25/12", "em 2 horas"; sem hora = 09:00.
- Recorrência diária, semanal, mensal ou anual, com antecedência ("2 dias antes").

**Caminhos comparados:**
- Gravar o lembrete como `remember`: só guarda o fato, ninguém avisa.
- Agendamento no celular (Termux): depende do device; o servidor é a fonte única e alcança todos os devices.
- Deixar o modelo calcular a data absoluta: erra o dia da semana (o modelo escreveu "qui 04/11" quando era quarta). Escolhemos o parser determinístico; o modelo só repete a data que **voltou** no resultado.

**Decisões de robustez:**
- `avisado_para` evita avisar duas vezes o mesmo vencimento.
- Nenhum device online: tenta de novo a cada ciclo por até 24 h depois do vencimento.
- O mensal guarda o **dia original**: 31 → 28/02 → 31/03, sem escorregar.
- Erro num ciclo não mata o loop.

**Medido no modelo real:**
- "me avisa em 2 minutos" → aviso saiu às **15:34:05**.
- Conta de luz mensal com 2 dias de antecedência: criada certa (ter 10/11, aviso no dia 08 às 09:00).
- **Bug achado:** o modelo mandava o campo como `action` (em inglês, por causa do nome da tool), e o listar falhava. Depois ele **inventou a lista de cabeça**. Fix: aceitar `acao` e `action`. Re-medido: listar e cancelar funcionaram.

**Pontos de atenção:**
- O servidor só sabe que *algum* device aceitou o `notify`: falta confirmar no celular que a notificação apareceu.
- Hora local do servidor (Windows): se o fuso mudar, os horários mudam junto.

**Melhorias:**
- Lembretes no painel (lista, editar, pausar).
- "Adiar 1 hora" na notificação.
- Registrar no cérebro quando a conta foi paga.

## 12. Mapa: datas preenchidas e horário nos pontos

**Escolha:**
- Os presets (1 h, 24 h, 7 dias, 30 dias) preenchem **De** e **Até** (Até = agora), mas o filtro continua "até agora" para a trilha seguir viva.
- Cada ponto ganha uma etiqueta de horário ("HH:MM", ou "dd/mm HH:MM" se a trilha cobre mais de um dia). **Só desenha as que não se sobrepõem na tela**: com zoom afastado aparecem poucas e, ao aproximar, aparecem as do meio. Recalcula a cada zoom ou arraste.
- Botão **Horários** liga e desliga; a escolha fica salva no navegador.

**Caminho comparado:** etiqueta em todos os pontos (vira borrão com centenas de pontos) contra seleção gulosa por distância em pixels. Escolhemos a seleção.

**Ponto de atenção:** não foi visto no navegador. Conferir o espaçamento (64×22 px) com uma trilha real.

## 13. Tutor e raciocínio (decisão provisória)

**Problema medido (A/B no modelo real):** com o raciocínio **desligado** (o padrão do CLI), o tutor **não chama ferramenta nenhuma**. Responde de cabeça e ainda escreve uma "fonte" que não leu. Isso já acontecia antes das mudanças de hoje. Com raciocínio: `search_docs` → `read_doc` → `log(abrir)` → resposta fiel.

**Escolha provisória:**
- No modo tutor o servidor **sempre raciocina**.
- O CLI mostra o raciocínio **colapsado** ao entrar no tutor e volta ao modo anterior ao sair. `/think off` dentro do tutor não é aceito, para a tela não mentir.

**Custo:** turnos do tutor de **20–50 s**, contra 5–17 s sem raciocínio.

**Caminhos comparados:**
- Busca prévia pelo servidor (buscar no acervo antes de chamar o modelo e injetar os trechos): **rejeitada pelo dono**. Frágil e sujeita a erro: a fala do aluno nem sempre é uma boa consulta, e o trecho errado injetado induz resposta errada com cara de fonte.
- Raciocínio forçado + CLI honesto: **escolhido por enquanto**.

**Próximo passo:** estudar outra forma de o tutor usar o material **sem depender do raciocínio** e sem busca prévia.

---

## Pontos de atenção gerais

- **Deploy:** editar no note → `scp` para o winserver → conferir SHA256 → `braza restart`. Antes de limpar o Windows, comparar os hashes de tudo que está diferente (hoje: nada existia só no servidor).
- **Windows ficou limpo em `e05246f` aguardando o pull.** Até o pull, um reinício do agente sobe código antigo. Depois do pull: `braza restart` **completo** (sem `--agent`), para conferir o SearXNG com o `settings.yml`.
- **Testes:** 136 passando. As 3 falhas antigas continuam: fake do stream em `test_llm_client`, `FakeOrch` em `test_voice_session` e `test_tokenizer` sem servidor.
- **O prompt sozinho não garante chamada de ferramenta no meio de outra tarefa** (registro do estudo, `log` do treino). O padrão que funcionou hoje: o gatilho é determinístico no servidor e o LLM só lê e extrai.
- **O modelo inventa quando a ferramenta falha** (lista de lembretes, tabela de comparação da web). Vale um guard do tipo "resposta afirma dado que nenhuma ferramenta devolveu".

## Melhorias em fila

1. Tutor sem depender do raciocínio (estudo novo, sem busca prévia).
2. Re-medir a pesquisa na web (citar só o que abriu; não completar tabela).
3. Lembretes no painel; confirmar a notificação no S24.
4. Acervo: renomear coleção e lista de coleções no envio.
5. Exp6 (ilustração do material do tutor) e live no celular: em andamento com o outro agente.
6. Consertar os fakes dos 3 testes antigos.
