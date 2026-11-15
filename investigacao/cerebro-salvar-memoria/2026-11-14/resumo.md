# Investigação — como o agente SALVA memória do dono

**Tema:** reescrita do cérebro (branch `feat/cerebro`), 1º passo = validar o comportamento de **salvar**.
**Data:** 2026-11-14 · **Modelo:** `qwen35-27b-abl` (vLLM iobraza:8002) · **Agente real** no winserver (C:\workspace\brazaia).
**Banco de provas:** `iobraza/docs/cerebro/pesquisa-salvar/` (gitignored).

> Documento vivo — o agente atualiza conforme a investigação anda.

---

## 1. Objetivo

Antes de escolher a ESTRUTURA do novo cérebro, entender **como o agente transforma a fala do dono numa memória** e **quando ele deve perguntar em vez de salvar**. Ex.: "fiz 4x10x50 no supino" → deve perguntar "50 kg é cada lado ou no total?" antes de gravar, e só salvar quando tiver certeza. Falas diferentes que dizem a mesma coisa ("10x50" = "dez vezes de 50") deveriam virar um registro buscável depois.

---

## 2. O MÉTODO (o maior aprendizado até aqui)

**Testar a tool SEMPRE dentro do agente real, nunca em script isolado.**

- Testes com prompt mínimo + 1 tool (thinking off) mostravam a tool "não disparando" / mangling de args → **artefato do ambiente pobre**. Com 1 tool só e prompt curto, o modelo erra o formato XML.
- No **agente real** (prompt completo + array de ~20 tools + parser `qwen3_coder` certo no servidor), a tool dispara **limpa**. O dono apontou: "as outras tools funcionam, então o problema é o teste isolado" — confirmado.
- Ferramenta de medição: `observar_agente.py` — manda cada fala por **HTTP** (`POST /api/v1/chat`) via proxy SOCKS5, **lê a sessão de volta** e classifica pelo `role:tool` (prova material do que executou):
  `salvou` · `perguntou` · `narrou` (respondeu sem chamar) · `erro` (guard de truncamento).
- Scripts isolados antigos (`observar.py`, `observar_fluxo.py`, `diag_*`) ficam como **histórico do erro de método**.

### Achado estrutural: tool > instrução
- **Prompt sozinho não faz o modelo perguntar.** Instrução suave e trava dura → perguntava 0/3 no ambíguo (sempre salvava o chute).
- Dar uma **tool `perguntar(pergunta)` separada** → passa a esclarecer o ambíguo. A decisão salvar-vs-esclarecer é **estrutura (tool)**, não regra de texto.

---

## 3. Iteração do PROMPT (think=false, 18 casos × 6 = 108 turnos)

| versão | mudança | salvou | erro | nota |
|---|---|--:|--:|---|
| **v1** | instrução base de memória | 38 | 11 | pouco salva |
| **v2** | "chamar é ação" + "perguntar é raro" + 3ª pessoa + "só o que disse" + *"salvar não tem limite"* | 50 | **22** | salva mais, mas erro DOBROU |
| **v3** | tira a linha *"salvar não tem limite"* | 48 | 12 | **melhor balanço** |

- **O "erro" NÃO é alucinação** — é o guard do orchestrator (`orchestrator.py:232`) quando o **argumento da tool é truncado** (estourou `max_tok` escrevendo o texto). A linha "salvar não tem limite" (v2) **primou** "limite" + encorajou args longos → erro dobrou. Tirá-la resolveu. **Lição: cuidado com negação no prompt — ela prima o indesejado.**

### Limites do prompt-só (think off) — o que resiste
1. **`narrou` ~23%** — o modelo diz "Anotado!" **sem** chamar `salvar`. É o "promete e não chama", estrutural; não some por texto.
2. **Calibrar o `perguntar` é instável** — v3 parou de perguntar o supino (cada-lado/total, 1/6) mas acertou o mercado (unidade, 6/6).
3. **Alguns nunca salvam** (sono, reunião, banco de dados) — pode ser o modelo julgando transitório/lembrete (talvez razoável).

---

## 4. EM CURSO — bateria com thinking LIGADO

**Hipótese:** com `think=True` o modelo raciocina "o dono contou um fato → preciso chamar salvar" antes de responder, e o `narrou` despenca. **Custo:** tokens/latência por turno.

- Script: `observar_agente_think.py` (cópia do v3, só `think=False→True`, saída em `agente_think.json/.md`).
- Sinal cedo (caso ambíguo supino, 3 reps): `perguntou · perguntou · salvou` — já pergunta mais que o v3.
- Resultado fechado (108 turnos) entra aqui quando terminar: **think-off (v3) × think-on** lado a lado em `salvou/perguntou/narrou/erro` + custo.

### Nota de infra (consertado hoje)
O agente **não tinha caído** (`braza status` = ONLINE). O `http=000` era **connect timeout curto** sobre o relay **DERP(sao)**: o winserver só tem rota por relay (sem conexão direta), 1ª conexão lenta. Fix no harness: `--connect-timeout 40` + `--socks5` com IP (sem depender de MagicDNS). vLLM tem rota direta e responde rápido.

---

## 5. Próximos passos (fila)

1. Fechar think-on × think-off e **decidir** o regime de salvar (off barato / on confiável / híbrido: thinking só no turno de salvar).
2. Resolver **consistência do texto salvo** (texto livre dá ~7 redações pra mesma coisa) → convenção canônica leve ou campos estruturados.
3. Alias do argumento (`texto`↔`text`) na tool real.
4. Só então: desenhar a **estrutura** do cérebro novo (entidades/busca).
