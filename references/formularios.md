# Formulários web (o que a importação XML não cobre)

A importação só cobre produções. Formação complementar, vínculos (PIBID,
estágio, monitoria) e eventos bloqueados como duplicado-por-título entram pelo
formulário. Todos são modal-iframe: mesmas técnicas de
`importacao.md` (eval atravessando iframe, neutralizar alert/confirm, campos
`disabled` precisam `e.disabled = false` antes de preencher).

## Cursos de curta duração (Formação Complementar)

Lista: `pkg_formacao_compl.lista`. Incluir:
`PKG_FORMACAO_COMPL.form?f_cod=<f_cod>&f_tipo=FA` (edição: `&f_seq=<n>`).

Passo 1: `f_nivel` (F=Curso de Curta Duração, 8=Extensão Universitária,
E=MBA, Y=Outros), `f_inst`, `f_curso`, `f_carga`, `F_STATUS`
(N=em andamento, S=concluído, I=incompleto) → Avançar → passo 2 (anos).

**Armadilha dupla**: `f_inst`/`f_curso` nascem `disabled` — habilite antes de
preencher, senão o POST sai sem os campos ("Campo Instituição obrigatório").
E a instituição exige CÓDIGO do registro: `f_cod_inst` (formato `V<13 dígitos>`
ou `D<...>`), `f_inst_old` = mesmo texto.

### Lookup de instituição

`gn_inst_form?f_cod=<f_cod>&f_nivel=<nível>` — busca por parte do nome
(`f_nome`) + `f_local_inst` (Brasileira/Estrangeira). O resultado vem num
`select[name=f_inst]` cujo option value é o código:
`V390600000006=Instituto Federal do Sertão Pernambucano /PE`.
`selecionar()` copia `f_cod_inst = value` (com prefixo) e limpa o curso.

- **Muitas empresas não existem no registro** (Alura, AOVS, DIO, Mimo, Canva
  não retornam nada). O resultado da busca tem botão **"Cadastrar Nova
  Instituicao"** (`gn_inst_outra?...&f_pgm=CURSO&f_nivel=...`) — fluxo manual
  na UI; por automação, prefira instruções ao usuário.
- Busque também pela razão social (Alura = AOVS Sistemas de Informática S.A.)
  e por nome completo ("Instituto Federal de Educação, Ciência e Tecnologia do
  Sul de Minas" acha IFSULDEMINAS; "IFSULDEMINAS" não acha).

### Lookup de curso + área

`prc_curso_form?f_cod=<f_cod>&f_pgm=FORM&f_nivel=<n>&f_inst=<código inst>` —
`select[name=f_curso]` com os cursos já registrados da instituição (value
`D90000042`). Curso novo: botão `novocurso()` →
`prc_curso_outro?...&f_sta_area=S` → `f_dsc_curso` (nome) + `f_area`
(**árvore hierárquica de áreas do conhecimento**, `prc_area_curso` — expansão
AJAX nível por nível; automação frágil, na dúvida deixe o usuário escolher a
área) → Confirmar gera o código.

Só depois: volte ao form de formação e submeta com `f_cod_curso` preenchido
(sem código o servidor devolve "Campo Curso obrigatório não informado").

## Eventos (quando o import bloqueia por título duplicado)

Itens vermelhos no importador não entram mais. Cadastre no form:
`PKG_EVENTO.form?f_cod=<f_cod>&F_STA_DIVULGACAO_CIENTIFICA=`.

- `F_FORMA` radios: 38=Convidado, 39=Participante, 40=Ouvinte — **clique
  (`.click()`), não só `.checked`**: o `select[name=F_TPO]` é populado no
  onclick (Participante: Poster/Painel, Apresentação oral, Outras formas).
  Depois force `sel.selectedIndex` e confira `sel.value` antes de salvar.
- `f_classif`: I=Internacional, N=Nacional, R=Regional, L=Local.
- `F_COD_PROD`: A31=Congresso, A32=Seminário, A33=Simpósio, A34=Oficina,
  A35=Encontro, A36=Olimpíada, A37=Feira.
- Campos: `f_nme_evento`, `f_ano`, `f_cidade`, `f_pais` (select ISO-3),
  `f_homepage`, `f_titulo` (título da apresentação/trabalho).
- A instituição promotora/participantes ficam numa etapa posterior do form —
  o cadastro básico salva sem ela ("body vazio" no POST = sucesso); confira na
  `pkg_evento.lista` e complete o que faltar pela UI.

## Vínculos (PIBID, estágio, monitoria, bolsas)

Atuação profissional: `pkg_ativ.lista` → linha da instituição
(`onclick` carrega `PKG_ATIV.cargos_vinculos?f_cod=...&f_codinst=V<código>&
f_seq_ativ=<n>`). Incluir: `PKG_ATIV.vinculos?...`.

Campos: `f_vinc` (tipo do vínculo, ex. "Bolsista PIBID"), `f_stavinc`
(select 0=Não/1=Sim — "possui vínculo empregatício"), `f_enqua`
(enquadramento, ex. "Monitor"/"Estagiário"/"Pesquisador" — **obrigatório**),
`f_carga` (horas semanais), `f_regime` (checkbox dedicação exclusiva),
`f_mes_ini`/`f_ano_ini`/`f_mes_fim`/`f_ano_fim`, `f_status` (N=Anterior
finalizado, S=Atual), `f_outras_inf` (textarea).

- Monitoria já catalogada = vínculo "Monitor" por período (padrão comum).
  Detalhe da disciplina cabe em `f_outras_inf`.
- Erro `ORA-12899 ... demasiado grande para a coluna` = algum campo estourou o
  limite do Oracle — reduza `f_vinc`/`f_outras_inf` ao mínimo e reenvie.
- A lista de vínculos aceita N vínculos por instituição (um por período/cargo).

## Conferência final (sempre)

Para cada seção mexida, abra a LISTA (`pkg_evento.lista`,
`pkg_formacao_compl.lista`, `pkg_ativ.cargos_vinculos...`) e conte/compare.
Mensagem de sucesso não é conferência; Exportar mostra a versão publicada, não
a atual.
