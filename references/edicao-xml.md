# Editar o XML com lattes-toolkit (sem poluir o currículo)

## Setup

```bash
npm i @paladini/lattes-toolkit        # v2.x
sudo pacman -S libxml2                # xmllint (opcional, recomendado)
```

Export CNPq sai em **ISO-8859-1**. Parse/serialize do toolkit preserva; se
editar com regex/Python, grave em iso-8859-1 com `errors="xmlcharrefreplace"`.

## Ciclo

```bash
npx lattes-toolkit parse <id>.xml --json          # resumo
npx lattes-toolkit diff original.xml novo.xml      # campos tipados alterados
npx lattes-toolkit validate novo.xml               # XSD via xmllint
```

Validação em 2 camadas — o importador usa o **DTD 2004** (LMPLCurriculo.DTD,
baixar de lmpl.cnpq.br), não o XSD 2022:

```python
from lxml import etree
dtd = etree.DTD(open("LMPLCurriculo.dtd", "rb"))
dtd.validate(etree.parse("novo.xml"))
for e in dtd.error_log: print(e.message)
```

O export do próprio CNPq tem 7 violações benignas no DTD 2004 (atributos novos
ORCID-ID, PCD, ELETRONICO, OUTRA-FORMA-DE-CONTATO, REDE-SOCIAL,
STATUS-OASISBR, ID-OASIS). **Toda violação além dessas 7 = o importador
recusa o arquivo inteiro.** Erros clássicos:

- `NATUREZA="ENSINO"` em PROJETO-DE-PESQUISA → não existe no DTD (use EXTENSAO
  /PESQUISA/DESENVOLVIMENTO/OUTRA).
- `&` sem `&amp;` em atributo gerado.
- Formação: as variantes têm ordem fixa em FORMACAO-COMPLEMENTAR
  (DE-EXTENSAO-UNIVERSITARIA*, MBA*, CURSO-DE-CURTA-DURACAO*, OUTROS*) —
  insira cada bloco após o último do MESMO tipo.
- Eventos: ordem fixa em PARTICIPACAO-EM-EVENTOS-CONGRESSOS
  (CONGRESSO, FEIRA, SEMINARIO, SIMPOSIO, OFICINA, ENCONTRO, EXPOSICAO,
  OLIMPIADA, OUTRAS-PARTICIPACOES...) — insira cada grupo após o último do
  mesmo tipo.
- ATUACAO-PROFISSIONAL: ordem fixa dos contêineres (VINCULOS, DIRECAO,
  PESQUISA, ENSINO, ESTAGIO, SERVICO-TECNICO, EXTENSAO, TREINAMENTO,
  OUTRAS-ATIVIDADES-TECNICO-CIENTIFICA, CONSELHO, PARTICIPACAO-EM-PROJETO).
- DADOS-COMPLEMENTARES: FORMACAO-COMPLEMENTAR, PARTICIPACAO-EM-BANCA-
  TRABALHOS-CONCLUSAO, ...-JULGADORA, PARTICIPACAO-EM-EVENTOS-CONGRESSOS, ...

## Regras de sequência

- `SEQUENCIA-PRODUCAO` é global (todas as produções): itens novos recebem
  números **maiores que o máximo do arquivo**, nunca lacunas (o servidor
  descarta em silêncio), nunca sem o atributo.
- `SEQUENCIA-FORMACAO`, `SEQUENCIA-FUNCAO-ATIVIDADE`, `SEQUENCIA-PROJETO`:
  idem, por seção.

## Onde cada coisa vai (locais que surpreendem)

- **Monitoria**: o XSD 2022 NÃO tem MONITORIA. Práticas que funcionam:
  (a) vínculo "Monitor" em Atuação Profissional › VINCULOS do período, ou
  (b) OUTRA-ATIVIDADE-TECNICO-CIENTIFICA com `ATIVIDADE-REALIZADA` descritivo.
- **PIBID/PIBIC**: PARTICIPACAO-EM-PROJETO › PROJETO-DE-PESQUISA com
  `NATUREZA="EXTENSAO"` (PIBID) / `"PESQUISA"` (PIBIC) — `ENSINO` quebra o
  import.
- **Estágio**: ATIVIDADES-DE-ESTAGIO › ESTAGIO com `ESTAGIO-REALIZADO`.
- **Eventos de atividades** (palestra/minicurso dentro de uma semana
  universitária): NÃO são cursos — vão como PARTICIPACAO-EM-CONGRESSO (ou
  SEMINARIO/ENCONTRO/OFICINA) com `TITULO` da atividade e
  `TIPO-PARTICIPACAO="Outras formas"`/`"Apresentação oral"`.

## Backup e conferência

- `npx lattes-toolkit set/patch` faz backup automático em `.lattes-backup/`;
  para edição por script, copie o original antes (`cp original.xml backup.xml`).
- Pós-edição: re-parse + validar DTD + comparar as 7 violações benignas como
  baseline. Qualquer erro novo → corrigir antes de importar.
- O importador só ACEITA o que valida; mas "valida" ≠ "importa": conferir o
  resultado na lista de cada seção.
