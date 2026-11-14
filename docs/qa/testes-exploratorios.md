# Testes exploratórios — método, cartas e registro

Responsável: João Pedro Araújo de Freitas Lyra (@Jadequilin) — QA de back-end e pipeline
Início: Sprint 4 · Rodada completa: Sprint 5

---

## 1. Por que, se a suíte está verde

A suíte automatizada do projeto tem 926 testes no back-end (98,81% de
cobertura, piso de 97%) e 436 no front. Ela responde bem a uma pergunta:
*o que a gente já sabe que pode quebrar continua funcionando?*

Ela não responde a outra: *o que a gente ainda não pensou em perguntar?*

Todo teste automatizado é uma pergunta escrita no passado. Ele encontra o
defeito que alguém previu. O defeito que ninguém previu passa por ele sem
encostar — e passa com a cobertura marcando 100% naquela linha, porque a
linha executou, só não foi interrogada.

A sessão piloto desta Sprint 4 mostra isso de forma literal. Os três achados
registrados na seção 5 estão em linhas com cobertura, exercitadas por testes
que passam. Um deles devolve uma credencial de longa duração no corpo de uma
resposta HTTP, e existe um teste no repositório que afirma que ela está lá —
porque quem escreveu o teste descreveu o comportamento observado, não o
comportamento desejado.

Teste exploratório é o método para fazer perguntas novas de forma organizada,
em vez de "mexer no sistema até achar alguma coisa".

## 2. Método: sessões com carta (SBTM)

Cada sessão tem:

**Carta** — uma missão de uma frase. Não é roteiro: diz o que investigar,
não quais passos dar. Quem executa decide o caminho durante a sessão.

**Tempo fechado** — 60 minutos, cronometrados. Acabou, acabou. O limite é o
que separa sessão de perambulação, e é o que torna o esforço comparável
entre sessões.

**Registro durante, não depois** — anotação do que foi feito, do que foi
observado e do que ficou por fazer, escrita enquanto a sessão corre.
Lembrança de meia hora atrás já perdeu os detalhes que importam.

**Relatório ao fim** — o formato está na seção 6.

**Divisão do tempo** — ao final, a estimativa de quanto da sessão foi
investigação da missão, quanto foi investigação de defeito encontrado no
meio do caminho, e quanto foi perdido em preparo de ambiente. Três sessões
seguidas com metade do tempo em preparo significam que o problema a resolver
é o ambiente, não o produto.

**Oráculo** — para cada observação, o critério que diz que aquilo é um
defeito: o requisito acordado com a cliente, o contrato da API, o
comportamento de uma tela equivalente, ou uma consequência que ninguém
defenderia se vista por escrito. Observação sem oráculo não vira achado,
vira opinião.

**O que vira teste automatizado** — todo achado confirmado entra na suíte
como especificação executável, no formato que a equipe já usa desde a Sprint
3: `xfail` estrito enquanto o defeito existe, teste verde quando o
comportamento correto chegar. O achado que não virar teste some na próxima
sprint.

## 3. Quando acontece

| Sprint | O que | Por quê |
|---|---|---|
| 4 | Método definido, cartas escritas, **sessão piloto EX-01 executada** | Provar o método com achado real antes de depender dele |
| 5 | Rodada completa: EX-02 a EX-10 | Exige o sistema com as histórias terminadas |

A rodada completa fica na Sprint 5 pela regra que a equipe combinou: a
Sprint 5 é só para o que depende de código que ainda não existe, nunca para
o que não deu tempo de fazer. Teste exploratório de uma funcionalidade pela
metade encontra, em maioria, a parte que falta — o que já se sabe. Encontrar
o que ninguém sabe exige o sistema inteiro de pé.

A sessão piloto não espera porque a superfície que ela investiga —
autenticação e integração com o Google — está pronta desde a Sprint 2.

## 4. Cartas

Prioridade por risco: consequência do defeito × chance de a suíte
automatizada não pegar.

### EX-01 — Superfície de autenticação e integração com o Google
**Executada na Sprint 4. Relatório na seção 5.**
Investigar **o que as respostas e os logs da autenticação carregam**,
procurando **segredo exposto e permissão concedida sem querer**.

### EX-02 — O caminho da cliente criando um administrador
Investigar **a criação de administrador pelo console do Keycloak, do jeito
que a cliente vai fazer**, procurando **estados em que o usuário é criado mas
não consegue entrar, ou entra com mais acesso do que deveria**.
Pontos de partida: papel de realm contra papel de client; `admin` sem papel
de setor (ver o teste verde em `tests/unit/test_achados_exploratorios.py`);
senha marcada como temporária; `Required user actions` pendente; perfil sem
sobrenome. Todos os cinco já apareceram em produção pelo menos uma vez.

### EX-03 — Evento de vários dias, virada de mês e fuso
Investigar **eventos que atravessam fronteiras de calendário**, procurando
**dia errado na tela, contagem regressiva errada e sumiço da listagem**.
Pontos de partida: 30/11 a 02/12; 31/12 a 02/01; evento de um dia só criado
às 23h; navegador em fuso diferente do de Brasília.

### EX-04 — Situação do evento e o que o público vê
Investigar **cancelar e adiar evento (US14)**, procurando **divergência entre
o que o painel registra, o que o site mostra e o que vai para o Google
Calendar**.
Pontos de partida: adiar para data no passado; cancelar evento já começado;
adiar duas vezes; cancelar evento com inscritos.

### EX-05 — Modalidade online e link de transmissão
Investigar **a modalidade do evento (US19)**, procurando **evento online
exigindo endereço, presencial exibindo link vazio, e link que não abre**.
Pontos de partida: online sem link; presencial com link preenchido; evento
online na tela de eventos próximos, que ordena por distância.

### EX-06 — Materiais para download
Investigar **o download de materiais (US11)**, procurando **link que leva à
visualização em vez do arquivo, arquivo sem permissão no Drive e tipo de
arquivo que o navegador abre em vez de baixar**.
Pontos de partida: PDF, imagem, planilha; arquivo com permissão restrita;
arquivo apagado do Drive depois de cadastrado; nome com acento e espaço.

### EX-07 — Galeria, fotos excluídas e o Drive
Investigar **a galeria do evento com a exclusão de fotos (US18)**, procurando
**foto excluída que reaparece, álbum vazio que quebra a tela e dessincronia
com o Drive**.
Pontos de partida: excluir todas as fotos; excluir e recadastrar a mesma
foto; álbum do Drive renomeado depois do cadastro.

### EX-08 — Busca de eventos
Investigar **a busca por nome (US01)**, procurando **resultado faltando por
acento, caixa ou grafia, e termo que derruba a consulta**.
Pontos de partida: `acampamento` contra `Acampamento` contra `acampamênto`;
termo de uma letra; só espaços; `%`, `_`, aspas; nome com hífen.
Observação: os 15 testes de endpoint da QA22 cobrem caractere especial
contra o Postgres real. Esta carta é sobre o que o visitante digita de
verdade, não sobre o que quebra a consulta.

### EX-09 — Eventos próximos e geolocalização
Investigar **a tela de eventos próximos**, procurando **o que acontece quando
não há localização**.
Pontos de partida: permissão negada; permissão ignorada (nem sim nem não);
GPS lento; usuário fora do Brasil; nenhum evento com coordenadas cadastradas.

### EX-10 — Celular de verdade e conexão ruim
Investigar **o site e o painel num aparelho físico com rede lenta**,
procurando **o que o emulador não mostra**.
Pontos de partida: toque em alvo pequeno; teclado virtual cobrindo campo;
imagem de 3,4 MB (`IDB_Jovem.gif`) em rede móvel; formulário longo do painel
em tela de 5 polegadas.
Observação: a suíte Playwright roda Pixel 7 e iPhone 13 **emulados**. Emulação
acerta o tamanho da tela e erra tudo o mais — latência, memória, teclado
nativo e precisão do toque.

## 5. Registro das sessões

### Sessão EX-01 — Superfície de autenticação e integração com o Google

| | |
|---|---|
| Carta | EX-01 |
| Data | 13/11/2026 |
| Duração | 60 min |
| Executou | João Pedro (@Jadequilin) |
| Build | back-end `a11325c`, front `c56c9a8` |
| Divisão do tempo | investigação da missão 60% · investigação de achado 35% · preparo 5% |

**Caminho percorrido.** Leitura do fluxo OAuth do Google de ponta a ponta
(`src/auth/controller.py`, `src/auth/service.py`), depois da verificação de
token e da matriz de papéis (`src/security.py`), e por fim conferência de
cada ponto suspeito contra a suíte existente, para saber se o comportamento
estava afirmado em algum teste.

**Achados.**

**EX-01-A — `/auth/callback` devolve o refresh token do Google no corpo da
resposta.** Severidade alta.
O endpoint retorna `"dados": tokens_google`, o payload inteiro vindo do
Google, com o `refresh_token` dentro. O `access_token` expira em uma hora; o
`refresh_token` não expira sozinho — quem o obtiver fala com a conta Google
do projeto (Drive, Calendar, Forms) até alguém revogar, e ninguém vai
revogar, porque não há sinal de que vazou.
Agrava: o callback é um `GET`, então a URL com o `code` já circula por
histórico, log de proxy e `Referer`.
Oráculo: credencial de longa duração não é dado de resposta. A própria
resposta denuncia a origem — `"instrucoes": "Copie o refresh_token do
terminal/tela"` — conveniência de desenvolvimento que foi para produção.
Registrado em: `tests/unit/test_achados_exploratorios.py`, `xfail` estrito.
Atenção ao corrigir: `tests/unit/test_auth_controller.py::test_callback_google_sucesso`
**afirma** hoje que os tokens voltam em `dados`, e vai precisar mudar junto.

**EX-01-B — o refresh token é impresso no log do contêiner.** Severidade alta.
`src/auth/controller.py` imprime o `refresh_token` em `stdout`, que em
produção é o log do contêiner: `docker compose logs jovem-backend` mostra a
credencial em texto puro, e o log sobrevive ao contêiner.
Oráculo: segredo que entra em log tem que ser considerado comprometido.
Registrado em: `tests/unit/test_achados_exploratorios.py`, `xfail` estrito.

**EX-01-C — administrador sem papel de setor alcança todos os setores.**
Severidade média, operacional.
`verificar_permissao_setor` libera o administrador que não tem nenhum papel
de setor. A regra está escrita e comentada como retrocompatibilidade, para
não trancar do lado de fora quem já era administrador antes da US03 — ou
seja, **está certo hoje**.
O risco é de operação, e cresceu nesta sprint: a cliente passou a criar
administradores sozinha pelo console do Keycloak. Criar o usuário com o
papel `admin` e esquecer o papel de setor não produz um administrador
limitado; produz um com acesso a tudo, sem nenhum aviso na tela.
Oráculo: a intenção da US03 é limitar por setor. Uma porta de
retrocompatibilidade que nunca fecha deixa de ser transição e vira a regra.
Registrado em: `tests/unit/test_achados_exploratorios.py`, **teste verde** —
trava o comportamento atual para que fechar a porta seja um ato deliberado.

**Ficou por fazer.** Rotação e revogação do refresh token exposto; verificar
se o token atual em produção é o mesmo que já passou por log; conferir o que
`GET /auth/login` registra.

## 6. Modelo de relatório de sessão

```
### Sessão EX-NN — <título da carta>

| | |
|---|---|
| Carta | EX-NN |
| Data | dd/mm/aaaa |
| Duração | 60 min |
| Executou | nome (@usuário) |
| Build | back-end <sha>, front <sha> |
| Divisão do tempo | missão N% · achado N% · preparo N% |

**Caminho percorrido.** O que foi feito, em prosa, na ordem em que aconteceu.

**Achados.**
**EX-NN-X — <uma frase>.** Severidade <alta|média|baixa>.
O que acontece. Por que é defeito (oráculo). Onde foi registrado.

**Ficou por fazer.** O que a sessão abriu e não fechou.
```

## 7. Severidade

| Nível | Critério |
|---|---|
| Alta | Perda ou exposição de dado, acesso indevido, ou funcionalidade central inutilizável |
| Média | Funcionalidade entrega resultado errado, com contorno possível |
| Baixa | Incômodo, inconsistência visual, texto |

Severidade é consequência, não esforço de correção. Defeito alto que se
corrige em uma linha continua alto.
