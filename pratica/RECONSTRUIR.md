# Como reconstruir um app com o REA e um agente de IA

Roteiro prático para usar o REA não só para *entender* um programa, mas para **fazer a sua própria versão** dele. Foi montado a partir de um relato real: um editor de vídeo pago (o tipo que corta gaguejos e enrolação de aulas) foi reconstruído por agentes em cerca de 4 a 5 horas, por uns US$ 20 de modelo, sem a pessoa escrever uma linha do app. A primeira versão já acertou aparência e função; o trabalho de verdade foi nos detalhes.

> ⚠️ **Área cinzenta.** Reconstruir o app de outra empresa pode violar licença e lei. Terreno seguro: programa seu, código aberto, software abandonado e uso pessoal para estudo. Não distribua o clone de um produto comercial. Veja os [Cuidados no README](../README.md#cuidados).

## 1. Escolha bem o alvo (o que dá e o que não dá)

A regra que mais pesa:

| Vem junto | Não vem |
|---|---|
| Tudo que roda **dentro do app**, na sua máquina: interface, edição, atalhos, abrir arquivos e imagens locais, exportar | Tudo que está **no servidor da empresa**: modelos e galerias na nuvem, menus que baixam conteúdo (ex.: WordArt, partes rápidas do Word), sincronização, contas, licenças |

Antes de começar, liste as funções que você usa e marque cada uma: **local** ou **nuvem**. Se o que você mais usa é nuvem, o clone vai decepcionar. No relato, o clone de um editor de texto digitava e carregava imagens bem, mas os menus que vinham do servidor ficaram vazios.

Bons candidatos: app de desktop antigo e estável, app Electron (o REA lê bem o `app.asar`), ferramenta em que **falta uma função** que você queria.

## 2. Monte o ambiente do agente

1. **Agente no terminal** (Claude Code, Codex ou o **dsh**, aberto, da seção 2b). Um *harness* modular, em que você liga e desliga funções como peças de Lego, ajuda: dê o repositório ao agente e peça "instale isso para mim".
2. **REA** instalado e conferido:
   ```bash
   npx rea-agents setup
   rea doctor --client claude_code --json
   ```
3. **Playwright** (o agente abre o clone, clica e compara com o original) e **busca na web** (documentação, formatos de arquivo). Peça em português simples: "instale o Playwright e use para testar a interface".
4. **Rode isolado.** Use a receita de [rodar-isolado.sh](rodar-isolado.sh): HOME de teste, teto de memória e de tempo. O alvo pode ser grande e travar a análise.

## 2b. Opção aberta e sem travas: dsh + modelo aberto

Claude, Codex e Gemini são ótimos, mas **às vezes recusam** partes de um trabalho de engenharia reversa, mesmo legítimo, e as regras deles podem mudar de um dia para o outro. A saída é montar o agente todo com peças abertas:

| Peça | O que é | Por que ajuda |
|---|---|---|
| **dsh** (`@deepseek-ai/dsh`, o *DeepSeek harness*) | O agente de código aberto da DeepSeek: lê e escreve arquivos, roda comandos, usa MCP (inclusive o REA) | É **aberto e não vem preso a um modelo**: você escolhe o modelo, local ou na nuvem. Quem impõe regras é o modelo, e esse quem escolhe é você |
| **Modelo aberto chinês** (Qwen, GLM, DeepSeek, Kimi, MiniMax) | Pesos abertos, rodam no seu Ollama ou em provedor barato | Recusam bem menos que os americanos em tarefas técnicas (engenharia reversa, segurança). Têm outras travas (temas políticos da China), que aqui não fazem diferença |
| **Versão "abliterated"** desses modelos | O mesmo modelo com o reflexo de recusar removido dos pesos | Não diz "não posso ajudar com isso" no meio do trabalho |
| **REA** | Os olhos do agente: Ghidra, análise de JS/Electron | Igual em qualquer agente |

**Como montar (local, nada sai da máquina):**

```bash
cd ~/projetos/dsh-sandbox && ./dsh local        # painel em http://127.0.0.1:9080
ollama pull huihui_ai/Qwen3.8-abliterated:27b   # confira a etiqueta "tools" na página do modelo
```

Depois peça ao dsh: *"instale isto e me ajude a usar para fazer engenharia reversa de um app: https://github.com/morluto/rea"*.

**Sem máquina para rodar o modelo?** Use uma plataforma pré-paga no estilo da OpenRouter, que hospeda esses modelos para você: coloca US$ 5 a 10 de crédito, pega a chave e pede ao dsh para integrá-la (no dsh-sandbox, `./dsh remoto`). As duas citadas no vídeo de referência:

- **[abliteration.ai](https://abliteration.ai)**: modelos abertos "abliterated", sem recusa, por API compatível com a da OpenAI (`https://api.abliteration.ai/v1`), em pré-pago por token ou plano mensal.
- **[OrcaRouter](https://orcarouter.ai)**: roteador com mais de 200 modelos num só endereço, igual à OpenRouter; tem versão aberta (MIT) que você pode hospedar.

Preços e catálogo mudam: confira na página antes de pôr crédito. Na nuvem, **o seu código sai da máquina**. Para app seu ou arquivo sensível, prefira o modelo local.

Passo a passo completo desta montagem, com prompts e os modelos conferidos: **[REA sem travas](https://inematds.github.io/rea-sem-travas/guia/)**.

> "Sem trava" quer dizer o modelo não recusar trabalho **legítimo** (app seu, interoperabilidade, estudo, segurança autorizada). Não é licença para quebrar proteção contra cópia, DRM ou licença, nem para redistribuir o que é dos outros.

## 3. Primeiro entenda, depois construa

Peça ao agente, nesta ordem:

1. **Mapa do app** com o REA: telas, módulos, formatos de arquivo, o que é local e o que chama a internet (`analyze-javascript-application` em app Electron; Ghidra para binário nativo).
2. **Lista de funções** com prova de onde cada uma está e o que ficou sem resposta (o REA devolve evidência e limites; não aceite "achismo").
3. **Plano da sua versão:** só as funções que você usa, mais as que faltam no original. Aprove o plano antes de o agente escrever código.

Para cada função do plano, o agente segue o mesmo método curto ([seção Método do guia](https://inematds.github.io/rea-engenharia-reversa/guia/#metodo)):

- **Uma função por vez.** "Como a busca acha a nota com três letras?", não "reverta o app".
- **Resumo primeiro.** Lê a visão geral da análise (`detail: "summary"`), abre um módulo e rastreia só aquela função. Não roda a mesma análise de novo: guarda o resultado.
- **Marca cada achado** como **visto** (nos arquivos, com trecho citado), **inferido** (dedução ainda sem prova) ou **desconhecido** (o REA não alcança, como o servidor). O desconhecido vira decisão sua: substituir, deixar de fora ou testar.
- **Ler o código não prova que ele roda assim.** O teste contra o original é que prova.

## 4. Construa a sua versão (não uma cópia)

- **Escreva do zero**, a partir do comportamento observado. Não copie código, ícones, marca nem textos do original.
- **Modelo para o trabalho pesado:** o relato usou um modelo barato e rápido para gerar o app (~US$ 20 em 4 a 5 horas). Para tarefas pequenas e repetidas (ex.: revisar transcrição), um modelo menor pela sua assinatura sai mais barato que pagar token por token.
- **Opções baratas além de Claude e Codex:** os modelos abertos chineses custam uma fração do preço e dão conta de muito código:
  - **DeepSeek** (o relato usou um harness do DeepSeek para montar o ambiente);
  - **GLM** (Zhipu / Z.ai; no relato, foi ele que executou as edições do vídeo);
  - **Qwen** (Alibaba);
  - **Kimi** (Moonshot);
  - **MiniMax**.

  Formas de usar: pela **OpenRouter** (uma chave só para todos e troca de modelo sem mudar o código), pela **abliteration.ai** ou **OrcaRouter** (seção 2b), pela API do próprio fabricante ou **local**, com os pesos abertos (Ollama), quando a máquina aguenta. Atenção: a API direta do fabricante manda seus dados para servidores na China. Para código ou arquivo sensível, prefira rodar local ou usar um provedor que hospede o modelo fora de lá.
- **Serviços prontos onde o original usava nuvem:** em vez de clonar o servidor, ligue um serviço que faz a mesma coisa. No editor de vídeo: transcrição (Whisper/Groq), áudio de estúdio (Auphonic), um modelo que assiste o vídeo quadro a quadro (Gemini Flash) e um modelo aberto (GLM) para decidir os cortes. Prefira o que roda local quando existir.
- **Aba de configurações** para as chaves desses serviços, guardadas só na máquina do usuário.

## 5. Itere nos detalhes

A primeira versão acerta o grosso. O resto é ida e volta:

- Peça ao agente para abrir **original e clone lado a lado** no Playwright, tirar print das duas telas e listar diferenças.
- Corrija **uma diferença por vez** (no relato, o botão de exportar levou várias rodadas).
- Teste com **um arquivo real seu** a cada rodada, não com exemplo inventado.

## 6. Vá além do original

O ganho não é ter a mesma coisa de graça: é ter **o que faltava**. No relato, o clone ganhou funções que o pago não tinha:

- clicar numa palavra da transcrição e o vídeo tocar dali;
- apagar palavras no texto e o vídeo pular esses trechos;
- lista de todos os cortes;
- "remover toda a enrolação" levou um vídeo de 9 para 6 minutos;
- legendas e download da transcrição.

E virou ferramenta de uso diário.

## Prompt pronto para o agente

```text
Quero fazer a minha própria versão do app <NOME>, para uso pessoal.
1. Use o REA para mapear o app em <CAMINHO>: telas, módulos, formatos
   de arquivo e o que roda local versus o que depende de servidor.
   Leia o resumo primeiro e rastreie uma função por vez. Marque cada
   conclusão como vista, inferida ou desconhecida, com a evidência.
2. Proponha um plano com só estas funções: <LISTA>, mais estas que o
   original não tem: <LISTA>. Para cada função de nuvem, sugira um
   serviço ou modelo local que a substitua. Espere eu aprovar.
3. Depois de aprovado, escreva o app do zero (sem copiar código, ícones,
   marca ou textos do original) e teste com o Playwright usando o
   arquivo <ARQUIVO-REAL>.
4. Compare com o original lado a lado, liste as diferenças e corrija uma
   por vez, me mostrando o print antes e depois.
```

## O que isso muda

Para muitos apps de desktop antigos, o **software deixa de ser a barreira**. O que separa um produto do outro passa a ser o **método e os dados**: o que fica no servidor, o conhecimento de quem usa, o cuidado no detalhe. Quando dá para construir quase tudo, a pergunta que importa é *o que vale a pena construir* e *com que cuidado*.

---

Mais conteúdo aberto e gratuito em **[inema.club](https://inema.club)**.
