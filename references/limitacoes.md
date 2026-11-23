# Limitações observadas (comprovadas em uso, nov/2026)

Consolidado de tudo que deu errado ou surpreendeu durante uma atualização real
(193 documentos locais, ~60 itens importados + formulários). Cada item foi
reproduzido; onde havia versão antiga da doc dizendo o contrário, anotado.

## 1. Importar XML

### Cobertura — só produções
O importador tem 17 módulos (artigos, livros/capítulos, trabalhos em anais,
apresentações, outras produções bibliográficas, softwares, produtos, processos,
trabalhos técnicos, outras produções técnicas, produção artística/cultural,
orientações, demais trabalhos, eventos, bancas). **Tudo fora disso no XML é
ignorado silenciosamente**: formação complementar (cursos), monitorias,
projetos de pesquisa, estágio, idiomas, áreas de atuação. Esses precisam de
formulário web (ver `formularios.md`) ou vínculo em Atuação Profissional.

### Validação pelo DTD 2004, não pelo XSD 2022
O arquivo INTEIRO é recusado ("Não foi possível processar o arquivo enviado...
DTD do Currículo Lattes") por qualquer violação de DTD:

- `PROJETO-DE-PESQUISA` com `NATUREZA="ENSINO"` → o DTD só aceita
  DESENVOLVIMENTO/EXTENSAO/PESQUISA/OUTRA. PIBID com natureza "ensino" precisa
  virar EXTENSAO/OUTRA.
- Atributos não declarados no DTD.
- `&` não escapado em texto/atributo.

O export oficial do CNPq carrega **7 violações benignas** (atributos novos que
o DTD 2004 não declara): `ORCID-ID`, `PCD` (DADOS-GERAIS); `ELETRONICO`,
`OUTRA-FORMA-DE-CONTATO`, `REDE-SOCIAL` (ENDERECO); `STATUS-OASISBR`,
`ID-OASIS` (MESTRADO). Elas importam; valide com lxml contra o
LMPLCurriculo.DTD e trate essas 7 como baseline — **violação nova ≠ 8**.

### Marcador vermelho BLOQUEIA
Item vermelho ("está cadastrada") confirmado no passo 3 responde "Não foi
possível atualizar as informações porque ocorreram os seguintes problemas:"
com **lista vazia** e consome a solicitação sem importar. A doc antiga da
comunidade dizia que vermelho era só aviso e duplicava — o comportamento atual
é bloqueio.

- O match é por **TÍTULO**: participação em evento cujo título iguala um resumo
  em "Outras produções bibliográficas" fica vermelha sem a participação estar
  cadastrada (falso positivo). Ex.: "Aprimorando o Ensino da Matemática..."
  (resumo) vs a apresentação do mesmo trabalho. Esses itens precisam de
  formulário.
- Itens "Produção sem título" (eventos de ouvinte) são sempre verdes e nunca
  casam — importá-los duplica os já existentes. Não os selecione.

### Filtro de ano do passo 2
`f_ano_prod` ("Produção C&T a partir de") LIMITA a lista do passo 3. O default
é o ano mais recente — com ele, itens de anos anteriores nem aparecem para
seleção. Selecione o MENOR ano da lista quando a edição tocar itens antigos.

### "Sucesso" não é conferência
O passo 3 pode responder "A operação foi realizada com sucesso!" e o item não
entrar. Caso reproduzido: evento sem a própria pessoa em
`PARTICIPANTE-DE-EVENTOS-CONGRESSOS` (descartado silenciosamente, consumindo
um número de sequência do servidor). Confira o resultado na LISTA da seção
(`pkg_evento.lista` etc.). O **Exportar entrega a versão publicada**, não a
atual — reexportar logo após importar mostra o currículo antigo e não prova
nada.

### Flakiness real
O mesmo item, no mesmo arquivo, pode não entrar numa tentativa e entrar na
seguinte (reproduzido 3×, também relatado por rlueder/lattes-editor PR #73).
Não trate "falhou" como definitivo em uma única tentativa; confira a lista e
reenvie.

### Sequências do servidor
`TRABALHO-EM-EVENTOS` sem `SEQUENCIA-PRODUCAO` = descarte silencioso; com
sequência em lacuna da exportação = descarte silencioso; acima do máximo =
aceito. Cada importação confirmada consome números do servidor (que não são os
do arquivo).

### Solicitação pendente
Solicitação não confirmada/descartada trava novos uploads: o Importar reabre
no passo 2/3 dela. Descarte (`f_ctr=D`) ou confirme antes de reenviar. Só
existe uma pendente por vez.

### Sessão
O cvlattesweb depende de SSO Keycloak: cookies `imp`/`imp2` copiados para curl
não autenticam (cai em `pkg_sec.logon` / redirect para lattes.cnpq.br). Toda
operação precisa ser dentro da aba logada (firefox-cli eval / fetch com
`credentials: "include"`). Form POST direto via curl está fora.

### ZIP
O passo 1 recusa ZIP — descompacte e envie o `.xml` puro.

### Erros opacos
"Mensagem de erro com lista de problemas VAZIA" e "Erro. Não foi possível
processar" não dizem a causa real (não há log acessível). Metodologia que
funcionou: bissecção — valide com lxml contra o DTD (a diferença de violações
contra o export original isola o problema), ou importe em lotes menores.

## 2. Formulários web

- **Campos `disabled`**: `f_inst`/`f_curso` (formação complementar) e outros
  nascem desabilitados e são populados por lookup. Setar `.value` sem antes
  `e.disabled = false` = o POST sai SEM o campo ("Campo Instituição obrigatório
  não informado" com o valor visível na tela).
- **Registro de instituições incompleto**: Alura, AOVS, DIO, Mimo, Canva não
  existem no lookup `gn_inst_form` (nem por razão social). Instituição nova
  exige "Cadastrar Nova Instituicao" (`gn_inst_outra`) + curso novo
  (`prc_curso_outro`) + área do conhecimento pela árvore AJAX
  (`prc_area_curso`) — inviável de automatizar com segurança; faça na UI.
- **Área do conhecimento**: árvore hierárquica com expansão AJAX; automação
  frágil e escolha de área é judgment do usuário (risco de poluir o registro).
- **`F_TPO` (tipo de apresentação) só popula no `onclick` do radio `F_FORMA`**:
  setar `.checked` não popula o select. Use `.click()` no radio e depois force
  `sel.selectedIndex` — e confira `sel.value` antes de salvar.
- **ORA-12899**: campos de vínculo/formação têm colunas Oracle pequenas;
  estourar devolve "valor é demasiado grande para a coluna" COM O NOME CORTADO.
  Reduza `f_vinc`/`f_outras_inf` ao mínimo e reenvie (bissecção de campos).
- **Erros DHTML**: a validação usa `alerta()` customizado (div na página), não
  `window.alert` — neutralizar `window.alert` não esconde, dá para LER a
  mensagem do DOM após `check()`. Aproveite isso para validar antes de salvar.

## 3. firefox-cli / automação de iframe

- **`confirm()`/`alert()` de iframe travam a thread da página**: o
  `firefox-cli eval` não retorna (timeout) e a extensão não vê o dialog
  (`dialog status` = handled:false). Recuperação: `firefox-cli reload`.
  Prevenção: sobrescreva `alert`/`confirm` no iframe E no top window antes de
  qualquer clique (`d.defaultView.alert = () => {}` etc.).
- **`input[type=file]` dentro de iframe**: `firefox-cli upload` não alcança;
  `snapshot` também não lista. Injete o arquivo via JS (gzip + base64 em 2
  evals + `DecompressionStream` — detalhes em `importacao.md`).
- **Limite do eval: 100 KB por script**; XML grande vai comprimido.
- **Mixed content**: a página https não faz fetch para `http://127.0.0.1`
  (NetworkError) — servidor local não serve de transporte.
- **Eval multi-statement pode retornar `undefined`** mesmo executando —
  embrulhe em IIFE com `return`.
- **Globals entre evals persistem, mas morrem em reload** — e o reload acontece
  sozinho em algumas telas; sempre resete os buffers (`window.__GZ = []`) antes
  de reusar, senão o base64 concatenado corrompe (`atob` → "invalid character").
- **Abas extras**: clicar em links de ajuda/DTD abre aba nova e o próximo eval
  vira "Ambiguous tab" — feche com `firefox-cli tab close id:<n>`.

## 4. lattes-toolkit

- `bibliographicProduction.booksAndChapters` não expõe
  `CAPITULO-DE-LIVRO-PUBLICADO` (só LIVRO-PUBLICADO-OU-ORGANIZADO) — capítulos
  existem no XML mas aparecem em outra chave/unmapped. Confira no XML cru antes
  de concluir que um capítulo "não está catalogado".
- O resumo do `parse` (articles/advisoriesCompleted) é raso para auditoria —
  para comparar "o que tem" vs "o que deveria ter", itere as estruturas tipadas
  ou o XML cru.
- `set`/`patch` já fazem backup e sincronizam só a seção — mas edição estrutural
  fina (ordem de contêineres, atributos de DTD) ainda é manual no XML cru
  (Python/lxml), seguida de `validate` + re-parse.
