# Impacto de Selenium e testes exploratórios no planejamento

Levantado em 13/11/2026 · João Pedro Araújo de Freitas Lyra (@Jadequilin)

---

## 1. O que mudou

O professor acrescentou duas exigências ao que já estava planejado:

1. testes que usem **Selenium**;
2. **testes exploratórios ao final**.

Nenhuma das duas substitui o que estava combinado para a Sprint 4. As duas
entram por cima, e cada uma cai numa sprint diferente — pelo motivo
explicado adiante.

## 2. Selenium: o erro que seria fácil cometer

O caminho óbvio é reescrever em Selenium o que a suíte Playwright já faz.
Seriam 436 testes reimplementados para fazer as mesmas perguntas, com
resultado idêntico e custo de manutenção dobrado. Volume, nenhum sinal novo.

A pergunta certa é outra: **o que a suíte atual, por construção, não
consegue verificar?** Três coisas, e as três são reais:

**A suíte Playwright simula a rede.** Todo teste de tela passa por
`tests/helpers/apiMock.js`. Nenhum teste do projeto hoje coloca o front
contra a API de verdade. Se o contrato da API mudar, a simulação continua
devolvendo o formato antigo e a suíte continua verde.

**Nada olha para o que está publicado.** A suíte roda contra um build
servido localmente. Entre 17/10 e 10/11 o Keycloak ficou fora do ar em
produção — **vinte e quatro dias** — e nenhuma verificação percebeu. O site
público continuou respondendo; só o login estava morto. A suíte estava verde
e estava certa: ela não fala com a produção.

**É uma pilha só.** Defeito escondido por configuração de uma ferramenta não
aparece nela. Uma segunda pilha de automação, independente, é verificação
cruzada de graça.

É aí que Selenium entra, e é por isso que ele **acrescenta** em vez de
duplicar: sem simulação nenhuma, apontado para onde a variável `URL_SITE`
mandar — o build no CI, a produção depois do deploy.

### O que foi entregue

`tests-selenium/` no repositório do front, em Python com pytest:

| Arquivo | O que faz |
|---|---|
| `test_entrega_do_build.py` | 7 testes + 1 especificação executável. Rodam contra qualquer instância |
| `test_historias.py` | 4 testes + 2 especificações executáveis, mapeados por história |
| `test_producao.py` | 6 testes que exigem a pilha real no ar. Só rodam com `ALVO=producao` |
| `servidor_api_falsa.py` | API de mentira que responde por HTTP de verdade |

### Cobertura por história

A suíte Playwright intercepta a rede dentro do navegador. Interceptar só
verifica chamada que alguém escreveu: **tela que nunca pede um dado não tem o
que interceptar**, então a ausência da funcionalidade inteira passa
despercebida. Por isso os testes de história recebem a resposta de um
servidor HTTP de verdade, com CORS e serialização reais.

| História | Verificação | Estado |
|---|---|---|
| US02 | Listagem sobrevive a evento sem coordenadas (o defeito da QA19, agora pelo lado do visitante) | verde |
| US19 | Selo de modalidade na lista, e filtro que preserva a ordem cronológica (CA5) | verde |
| US01 | Busca por nome, sem acento e sem caixa | verde |
| US14 | Evento cancelado aparece como cancelado | `xfail` estrito |
| US11 | Página pública de materiais lista os downloads | `xfail` estrito |

### O que os dois `xfail` revelaram

**US10, US11 e US14 não têm a ponta do front na `main`.** O back-end está
completo desde a Sprint 3 — migrações `b5e2c8d4a1f7`, `c8f3a6e2d9b4` e
`d4a7b9c1e3f6`, rotas `/material/`, `/painel/links` e
`/evento/{id}/situacao` publicadas no OpenAPI de produção. No front da
`main`, a palavra `situacao` não aparece nenhuma vez em `src/`, e nenhum
serviço chama `/material/` ou `/painel/links`.

O trabalho existe: está na branch `feat/us10-us11-us14` (commit `ec63ae0`,
1 761 linhas, com a página pública `/materiais`), que **nunca foi integrada
à `main`**. Em produção, hoje:

- evento cancelado aparece na listagem pública igual a qualquer outro, com
  data e local, sem nenhum aviso — alguém pode viajar para um evento que não
  vai acontecer;
- não há página de materiais;
- não há links fixados no painel.

Nenhuma suíte reclamou disso, e as duas estavam certas: a do back-end testa
o back-end, e a do front intercepta chamadas que o front não faz.

**Consequência para o planejamento:** integrar `feat/us10-us11-us14` é
trabalho de Sprint 4, antes de qualquer item novo, e o registro da Sprint 3
precisa ser conferido — se essas três histórias constam como completas nas
duas pontas, o número está errado.

### Uma divergência menor, registrada

`searchEvents` — a função que chama `/evento/buscar`, o endpoint com
similaridade por trigrama e 15 testes de endpoint contra o Postgres real
(QA22) — é chamada de **um único lugar em todo o front: a página de arnês
`/test-coverage`**. Nenhuma tela a usa.

A busca que o visitante usa na página de eventos é local, em JavaScript,
sobre a lista já carregada. As duas funcionam, mas são implementações
diferentes com comportamentos diferentes, e a que o usuário alcança é a que
não tem teste de back-end. Não é defeito; é decisão a tomar.

O que esses testes verificam e a suíte Playwright não verifica:

- **Rota profunda entra direto sem cair no 404 do servidor.** Numa SPA,
  `/eventos` não é arquivo: depende da regra de `rewrites` do `vercel.json`.
  É configuração de servidor, e configuração de servidor é o que muda entre
  o build local e a produção.
- **Navegação do menu não recarrega a página inteira.**
- **O painel exige sessão com o navegador limpo**, e não com o estado de
  autenticação montado pela ferramenta (`tests/helpers/adminAuth.js`).
- **O `index.html` publicado declara idioma e viewport.**
- **O Keycloak do realm responde** — o incidente dos 24 dias, virado teste.
- **A API libera o domínio do site no CORS.** CORS errado não aparece como
  erro na tela: aparece como lista vazia, e parece que não há evento
  cadastrado. Por isso essa verificação é por HTTP direto, não pelo navegador.
- **As migrações da Sprint 3 chegaram ao banco de produção.** Deploy com
  `git pull` e `build` mas sem `alembic upgrade head` deixa o código novo
  falando com o banco velho; o sintoma é 500 na primeira consulta que toca a
  coluna nova, nunca erro de build.
- **Eventos saem da API e chegam à tela**, com dado real e nada simulado no
  meio.

Mais uma especificação executável, no formato que a equipe usa desde a
Sprint 3 (`xfail` estrito):

- **A página `/test-coverage` não pode ir para produção.** Ela é o arnês que
  a suíte Playwright usa para medir cobertura, e hoje está publicada na rota
  pública, expondo cinco serviços em `window` — entre eles `window.api`, o
  cliente HTTP já configurado com o interceptador que anexa o token da
  sessão. O peso no bundle é pequeno (956,57 kB → 943,22 kB, medido); o
  problema é exposição, não tamanho. O conserto não é apagar a página, que a
  suíte Playwright usa, e sim condicionar a rota a `import.meta.env` para
  que ela não entre no build de produção.

### No pipeline

- **Job `selenium` no CI**, depois do `build`, consumindo o mesmo artefato
  `dist` que o job de E2E. Roda em Chrome, que já vem no runner.
- **Workflow `verificacao-producao.yml`**, separado: manual pelo botão *Run
  workflow* depois de cada deploy, e automático toda segunda às 9h17 de
  Brasília, para que uma queda silenciosa apareça em dias e não em semanas.

Firefox: a suíte roda nele com `NAVEGADOR=firefox` e é assim que a
verificação de compatibilidade é feita à mão — a Playwright cobre Chromium e
WebKit, nunca Firefox. Ficou fora do CI de propósito, por ora: portão que
nunca passou uma vez é portão que fica vermelho na primeira PR de outra
pessoa, e aí alguém desliga o portão. Entra depois de uma execução manual
limpa.

## 3. Exploratórios: por que a maior parte fica na Sprint 5

O professor pediu "ao final", e isso coincide com a regra que a equipe já
combinou: **a Sprint 5 é só para o que depende de código que ainda não
existe, nunca para o que não deu tempo de fazer.**

Teste exploratório de funcionalidade pela metade encontra, em maioria, a
parte que falta — o que já se sabe. Encontrar o que ninguém sabe exige o
sistema inteiro de pé. É o caso mais legítimo possível de item de Sprint 5.

Mas entregar na Sprint 5 um método que nunca foi exercitado é apostar. Por
isso a divisão:

- **Sprint 4** — método definido, 10 cartas escritas e **uma sessão piloto
  executada**, sobre a superfície que está pronta desde a Sprint 2.
- **Sprint 5** — rodada completa, cartas EX-02 a EX-10.

O método e as cartas estão em `docs/qa/testes-exploratorios.md`.

### O que a sessão piloto encontrou

Três achados, nenhum deles visível para a suíte automatizada — todos em
linhas **com cobertura**, exercitadas por testes que passam:

| Achado | Severidade |
|---|---|
| `/auth/callback` devolve o refresh token do Google no corpo da resposta | Alta |
| O refresh token é impresso no log do contêiner | Alta |
| Administrador sem papel de setor alcança todos os setores | Média, operacional |

O primeiro é o mais ilustrativo do método: existe hoje um teste no
repositório que **afirma** que os tokens voltam na resposta
(`test_auth_controller.py::test_callback_google_sucesso`). Quem o escreveu
descreveu o comportamento observado, não o desejado. Cobertura não reclama
disso; só alguém perguntando o que a resposta carrega.

Os três estão registrados em `tests/unit/test_achados_exploratorios.py`: os
dois primeiros como `xfail` estrito, o terceiro como teste verde que trava o
comportamento atual.

## 4. Como fica o planejamento

A numeração abaixo continua de onde a Sprint 3 parou (QA18–QA24 meus,
QA25–QA31 do Rivadalvio) e precisa ser conferida contra o documento de
planejamento antes de publicar.

### Sprint 4 — acrescentado

| Item | Entrega |
|---|---|
| QA32 | Suíte Selenium: 11 testes verdes e 3 especificações executáveis, cobrindo US01, US02, US11, US14 e US19 |
| QA33 | Verificação de produção: 6 testes, job no CI e workflow pós-deploy |
| QA34 | Programa de testes exploratórios: método, 10 cartas, modelo de relatório |
| QA35 | Sessão EX-01 executada: 3 achados, registrados como teste |

### Sprint 4 — o que já estava e continua

Back-end da US22; CA2 e CA3 da US03; testes de tela das histórias que se
completaram no fim da Sprint 3.

### Sprint 4 — o que entra na frente de tudo

Integrar `feat/us10-us11-us14` à `main` e publicar. São três histórias com o
back-end pronto e em produção há semanas, sem a ponta que o usuário usa.

### Sprint 5

| Item | Entrega |
|---|---|
| QA36 | Rodada completa de exploratórios: cartas EX-02 a EX-10, com relatório por sessão |

Mais o que já estava reservado por depender de código que ainda não existe.

## 5. O que isso custa

A suíte Selenium roda em 12,3 segundos contra o build local, em 20 testes. O
job no CI acrescenta um `setup-python`, um `pip install` e um segundo `npm
run build` (0,9 s) ao que já existe — nenhuma mudança no caminho crítico,
porque ele roda em paralelo com o job de E2E, que leva muito mais.

A rodada exploratória da Sprint 5 são 9 sessões de 60 minutos: nove horas de
trabalho, divisíveis entre os dois QAs.

## 6. O que fica em aberto

- **Integrar `feat/us10-us11-us14`.** Três histórias sem a ponta do front em
  produção. Os dois `xfail` da suíte Selenium viram verdes no dia da
  integração, e aí os marcadores saem.
- **Decidir sobre as duas buscas.** Ou a tela passa a usar `/evento/buscar`,
  ou o endpoint sai — manter os dois é manter uma divergência silenciosa.
- **Rotacionar o refresh token do Google.** O token em produção hoje já
  passou por log. Enquanto não for revogado e trocado, a correção do
  EX-01-A e do EX-01-B conserta o vazamento futuro e não o passado.
- **Tirar a rota `/test-coverage` do build de produção**, sem quebrar a
  suíte Playwright que depende dela.
- **Firefox no CI**, depois de uma execução manual limpa.
