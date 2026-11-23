# Exportar e Importar o XML pela sessão do Firefox

## Exportar

O export fica em "Menu secundário › Exportar" (`PKG_MENU.menu`). O modal carrega
um iframe `pkg_impcv.get_arquivo?f_cod=...` com o formulário:

```js
// firefox-cli eval — dentro da página do menu
(() => {
  const f = [...document.querySelectorAll("iframe")].find(f => (f.getAttribute("src")||"").includes("pkg_impcv"));
  const fm = f.contentDocument.PKG_IMP;
  fm.f_tpo_arq[1].checked = true;   // [0]=RTF, [1]=XML
  fm.f_ctr.value = "G";
  fm.submit();
})()
```

O download cai em `~/Downloads/CV_<id>.zip` (contém `<id>.xml`, ISO-8859-1).

**Não tente curl/fetch com os cookies** (`imp`, `imp2`): a sessão do cvlattesweb
depende do SSO Keycloak (`login.cnpq.br`) e o curl cai em `pkg_sec.logon`.

## Importar — os 3 passos

Menu secundário › Importar (`pkg_importa_xml.inicio?f_cod=<10 chars>`).

**Passo 1** — form `fl`, POST multipart para `pkg_importa_xml.inicio`:
`file` (o .xml puro, NÃO o zip), `f_cod`, `f_ctr=I`, `f_ret`. O botão chama
`check()` que só valida extensão `.xml`.

**Passo 2** — form `pkg_imp_xml`, POST urlencoded para
`pkg_importa_xml.mostra_modulos`: `f_cod`, `f_solic` (nº da solicitação criada
no passo 1), 17 caixas `f_chkp` (1 Artigos completos … 16 Eventos, 17 Bancas) e
o filtro **`f_ano_prod`** ("Produção C&T a partir de") — que LIMITA a lista do
passo 3. Para não perder itens antigos, mande o MENOR ano disponível no select.

**Passo 3** — form `pkg_imp_xml`, POST para `pkg_importa_xml.mostra_lista_prod`:
um checkbox `f_sq` por item (o valor é id DO SERVIDOR, não o
`SEQUENCIA-PRODUCAO` do arquivo), marcador `<b class="quadrado verde">`
(não cadastrada) ou `vermelho` (já cadastrada). `enviar()` põe `f_ctr=I`.

## Armadilhas observadas (testado na prática)

- **Itens vermelhos são BLOQUEADOS** na versão atual: confirmar item vermelho
  responde "Não foi possível atualizar as informações porque ocorreram os
  seguintes problemas:" com lista VAZIA e consome a solicitação sem importar
  nada. (A doc antiga dizia que vermelho era só aviso — mudou.) O match do
  marcador é por TÍTULO: uma participação em evento cujo título iguala um
  resumo em "Outras produções bibliográficas" fica vermelha e não entra.
  Nesses casos, cadastre pelo formulário (ver `formularios.md`).
- O passo 1 recusa o arquivo inteiro com "Não foi possível processar o arquivo
  enviado... DTD do Currículo Lattes" para qualquer violação de DTD — inclusive
  `NATUREZA="ENSINO"` em `PROJETO-DE-PESQUISA` (o DTD só tem
  DESENVOLVIMENTO/EXTENSAO/PESQUISA/OUTRA) e atributos não declarados novos.
  Valide com lxml contra o DTD LMPLCurriculo.DTD (o original do CNPq tem 7
  violações próprias — "No declaration for attribute ORCID-ID/PCD/ELETRONICO/
  OUTRA-FORMA-DE-CONTATO/REDE-SOCIAL de DADOS-GERAIS/ENDERECO e
  STATUS-OASISBR/ID-OASIS de MESTRADO" — o export do CNPq sai assim e importa;
  essas sete são benignas).
- **Uma solicitação pendente trava novos uploads**: se o Importar abre no
  passo 2/3, ou Descarte a pendente (`f_ctr=D`) ou confirme-a antes de reenviar.
- "Existe solicitação de Importação de XML pendente" ao reabrir o Importar =
  solicitação ainda em aberto (passo 2/3).
- Depois do Confirmar do passo 3 a solicitação é consumida e a página pode
  travar em alert nativo (a extensão não vê dialogs de iframe). Deu timeout no
  eval → `firefox-cli reload` e confira o RESULTADO na lista do item, nunca na
  mensagem.

## Dirigindo os passos com firefox-cli

- O conteúdo dos modais vive em IFRAMES — `snapshot` não alcança inputs de
  arquivo; use `eval` atravessando `document.querySelectorAll("iframe")`.
- `firefox-cli upload` não acha `input[type=file]` dentro do iframe. Injete o
  arquivo por JS: fetch de um servidor local falha (mixed content); use gzip +
  base64 (o XML de ~170 KB comprime para ~24 KB de base64, cabe no limite de
  100 KB do eval) e `new File([bytes], "x.xml")` + `DataTransfer`:
  ```js
  window.__GZ = "<base64 do gzip>";  // 1º eval
  (async () => {                      // 2º eval
    const bytes = Uint8Array.from(atob(window.__GZ), c => c.charCodeAt(0));
    const xml = await new Response(new Blob([bytes]).stream()
      .pipeThrough(new DecompressionStream("gzip"))).arrayBuffer();
    const f = [...document.querySelectorAll("iframe")]
      .find(f => (f.getAttribute("src")||"").includes("pkg_importa"));
    const d = f.contentDocument;
    d.defaultView.alert = () => {}; d.defaultView.confirm = () => true;
    const dt = new DataTransfer();
    dt.items.add(new File([xml], "curriculo.xml", { type: "text/xml" }));
    d.querySelector("input[type=file]").files = dt.files;
    setTimeout(() => d.querySelector("input[name=bt]").click(), 80);
  })()
  ```
- **Sempre neutralize `alert`/`confirm` no iframe E no top window ANTES de
  clicar** — `confirm()` de iframe trava a thread e o `firefox-cli eval` só
  volta após `firefox-cli reload`.
- Checkbox `f_sq` do passo 3: marque por linha inteira (o checkbox vive dentro
  de `table.int > tbody > tr` aninhado; filtre `table.int tr` com
  `b.quadrado` + `input[name=f_sq]`).
- Abas extras abrem com a DTD etc. (`firefox-cli tab close id:<n>`); se o eval
  reclamar de aba ambígua, passe `--tab id:2`.
