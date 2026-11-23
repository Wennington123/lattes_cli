# lattes_cli

Skill de automação do [Currículo Lattes](https://lattes.cnpq.br/) para agentes
de código: atualiza o currículo usando a **sessão real do Firefox**
(via [firefox-cli](https://github.com/respawn-llc/firefox-cli)) e o
[lattes-toolkit](https://github.com/paladini/lattes-toolkit) para editar o XML
localmente.

## O que faz

1. Exporta o XML do Lattes pela sessão logada do Firefox
2. Cataloga certificados/documentos locais (PDF/OCR) e compara com o XML
3. Edita o XML com validação DTD+XSD (lattes-toolkit)
4. Importa o XML de volta (produções, eventos, bancas)
5. Preenche formulários web para o que a importação não cobre (cursos,
   vínculos, eventos bloqueados como duplicado)

## Instalação

Pré-requisitos: Firefox com a extensão firefox-cli + native host
(`firefox-cli setup && firefox-cli doctor`), sessão logada no Lattes, Node 18+.

```bash
# skill (Command Code / Claude / Codex — formato agent-skills)
npx skills@latest add Wennington123/lattes_cli

# toolkit usado pela skill
npm i @paladini/lattes-toolkit
```

## Estrutura

| Arquivo | Conteúdo |
| --- | --- |
| `SKILL.md` | Fluxo em 5 passos + limites duros da plataforma |
| `references/importacao.md` | Exportar/importar o XML: os 3 passos do Importar, filtros, armadilhas, injeção de arquivo |
| `references/edicao-xml.md` | Edição do XML com lattes-toolkit: validação DTD 2004 + XSD 2022, sequências, onde cada coisa vai |
| `references/formularios.md` | Formulários web: formação complementar (lookups de instituição/curso/área), eventos, vínculos |

## Licença

MIT
