---
name: lattes_cli
description: >
  Atualiza o Currículo Lattes automaticamente usando firefox-cli (sessão real
  do Firefox) + lattes-toolkit (edição local do XML). Exporta o XML, compara
  com certificados/documentos locais, importa produções/eventos/bancas e
  preenche formulários que o XML não cobre. Use para "atualizar meu lattes",
  "importar certificados no lattes", "lançar eventos no lattes", "corrigir
  itens catalogados no lugar errado".
---

# Lattes via CLI

Fluxo completo para atualizar o Currículo Lattes sem digitar formulários um a um.
Depende de: `firefox-cli` (extensão + native host instalados, `firefox-cli doctor`)
com sessão logada no Lattes, e `npm i @paladini/lattes-toolkit` (Node 18+).

## O fluxo em 5 passos

1. **Exportar o XML** pela sessão do Firefox (não baixa ZIP pela curl — a sessão
   é SSO e o curl não autentica). Navegue até o menu e submeta o formulário de
   exportação (radio XML + `f_ctr=G`) dentro do iframe; o download cai em
   `~/Downloads/CV_<id>.zip`.
2. **Catalogar os documentos locais** (certificados etc.): `pdftotext -l 3` em
   lote; PDFs sem texto → `pdftoppm -r 150` + tesseract (WASM via
   `tesseract.js` se o binário não existir). Compare com o XML parseado.
3. **Editar o XML** com lattes-toolkit e validar (ver `references/edicao-xml.md`).
4. **Importar o XML** na plataforma (ver `references/importacao.md`).
5. **Formulários** para o que a importação NÃO cobre (ver
   `references/formularios.md`): cursos de formação complementar, vínculos,
   eventos bloqueados como duplicado-por-título.

## Limites duros da plataforma (aprendidos testando)

- O Importar só traz **produções** (17 módulos: artigos, livros, anais,
  apresentações, softwares, eventos, bancas...). **NÃO importa** formação
  complementar, monitorias, projetos, estágio, idiomas — esses vão por
  formulário, ou ficam como vínculos em Atuação Profissional.
- O importador valida contra o **DTD antigo** (não o XSD 2022): a natureza de
  projeto `ENSINO` é rejeitada — use `EXTENSAO`/`PESQUISA`/`OUTRA`.
- Evento sem a própria pessoa em `PARTICIPANTE-DE-EVENTOS-CONGRESSOS` pode ser
  descartado silenciosamente.
- "Sucesso" na tela não garante que o item entrou: confira sempre na LISTA do
  item (ex.: `pkg_evento.lista`) — o Exportar entrega a versão publicada, não a
  atual.
- Item vermelho ("já cadastrada") no passo 3 é **bloqueado** com erro vazio, e
  o match é por título (falso positivo com resumos em outras produções).
- Importação é **flaky**: o mesmo item pode falhar e entrar na tentativa
  seguinte. Confira a lista e reenvie.
- Instituições comuns (Alura, DIO, Mimo, Canva) **não existem no registro** do
  CNPq — cursos delas exigem cadastro de instituição/curso/área na UI.
- Erros da plataforma são opacos ("problemas:" com lista vazia); valide o XML
  com lxml contra o DTD e use bissecção de lotes.
- Automação de iframe: neutralize `alert`/`confirm` ANTES de clicar, campos
  `disabled` precisam `e.disabled = false`, `confirm` de iframe trava o eval
  (recupere com `firefox-cli reload`).

Lista completa com reproduções e recuperações: `references/limitacoes.md`.
Detalhes operacionais: `references/importacao.md`, `references/edicao-xml.md`,
`references/formularios.md`.
