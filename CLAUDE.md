# Painel de Vendas — diretrizes do projeto

Painel web de vendas imobiliárias (vendas, estoque, corretores, clientes e fluxo de caixa) montado a partir de planilhas Excel.

- **Repositório:** https://github.com/filipexandrade/salesdashboard (branch `main`)
- **Publicado em:** https://filipexandrade.github.io/salesdashboard/ (GitHub Pages, raiz da `main`)
- **Pasta local:** `OneDrive\Área de Trabalho\SALES DASHBOARD - CLAUDE`, que é um clone do repositório

## Arquivos

| Arquivo | Papel |
|---|---|
| `index.html` | O painel inteiro, em um só arquivo: HTML, CSS e JavaScript, sem etapa de build |
| `modelo-painel-vendas.xlsx` | Planilha de vendas (fonte principal dos dados) |
| `Cadastro de Unidades.xlsx` | Cadastro de todas as unidades (alimenta o Estoque) |
| `robots.txt` | Bloqueia indexação por buscadores (`Disallow: /`) |
| `logo-egecon.png` | Logo da Egecon exibida no título do painel |
| `teste-layout.html` | Cópia local para testar mudanças de visual. Não vai para o GitHub |
| `.claude/launch.json` | Servidor local para prévia (`python -m http.server 8765`). Não precisa ir ao GitHub |

## Arquitetura

### Página única, sem dependências locais
- Tudo vive no `index.html`. Bibliotecas externas: SheetJS `xlsx@0.18.5` (cdnjs) para ler e gerar planilhas, e a fonte Inter (Google Fonts).
- O JavaScript está em uma IIFE com `"use strict"`, organizado em seções comentadas (`/* ---------- Nome ---------- */`): Formatação, Leitura da planilha, Estado, Abas, Filtros, Render geral, Tabelas, Página 2, Página 3, Gráficos, Persistência, Importação, Cadastro, Modelo, Exemplo, Tema, Início.
- Os gráficos são SVG desenhados à mão (`drawBar`, `drawLine`, `drawFx`). Não há biblioteca de gráficos.

### Origem dos dados (ordem de prioridade)
1. **Planilhas do repositório:** ao abrir, `carregaRepo()` busca via `fetch` os arquivos ao lado do `index.html`:
   - vendas: `vendas.xlsx`, `Vendas.xlsx` ou `modelo-painel-vendas.xlsx`
   - cadastro: `cadastro.xlsx`, `Cadastro.xlsx`, `Cadastro de Unidades.xlsx` ou `cadastro-de-unidades.xlsx`
   - **Para atualizar o painel publicado, basta substituir a planilha no repositório com o mesmo nome e dar push.**
2. **Banco compartilhado** (`window.claude.use("db")`), só quando a página roda como Artifact do Claude.
3. **Cópia local** no `localStorage` (`painel-vendas-v2`).
4. **Cópia embutida:** o bloco `<script type="application/json" id="embed">` guarda um retrato dos dados (de 19/10/2026), usado se as planilhas não puderem ser lidas.

> O `<body>` do `index.html` foi salvo já renderizado (com tabelas e SVGs preenchidos). O JavaScript redesenha tudo ao carregar. Ao editar a marcação, mexa na estrutura (ids, classes) e não no conteúdo gerado.

### Leitura das planilhas
- `readWorkbook()` percorre todas as abas e acha a linha de cabeçalho nas 30 primeiras linhas.
- As colunas são reconhecidas por apelidos (`ALIAS`), sem diferenciar maiúsculas, acentos ou espaços. Ex.: "Valor Venda", "valor da venda" e "Preço de venda" viram `valor`.
- Abas esperadas no modelo: **Unidades** (vendas), **Fluxos específicos**, **Cadastro de Unidades**, **Corretores** e **Instruções**. O botão "Baixar modelo" gera essa planilha (`baixarModelo()`).
- O cadastro também pode ser importado com mapeamento manual de colunas (janela `#cadModal`), e o formato fica salvo como modelo (`painel-modelo-cadastro`).

### Estado e regras de negócio
- O estado global é `S`. `setData()` converte o payload compacto (`u`, `s`, `k`, `c`, `f`, `d` = unidades, vendas, cadastro, corretores, fluxos, despesas) e chama `derive()`.
- Unidades são casadas entre abas por `ukey(emp, unid)`, que ignora maiúsculas, acentos, espaços e hífens e trata "Bloco" como "BL".
- **Permutas e reservas administrativas não entram nos totais de venda.** Aparecem à parte nos indicadores e na tabela de vendas.
- Estoque = cadastro menos vendidas, permutas e reservas administrativas.
- Os filtros (ano, empreendimento, tipo, corretor) ficam em `S.f`. `render()` redesenha a aba ativa.

### Fluxo de caixa (aba 3)
- Mostra só o empreendimento **Andrade Home Resort** (`FX_EMP`). Nessa aba, o filtro de empreendimento tem apenas essa opção. O filtro escolhido nas outras abas é preservado ao voltar.
- Alerta `#fxAlert` acima do gráfico: mês em que o caixa acumulado fica negativo pela primeira vez (`CS.negYm`, calculado em `cashSeries`), o pico e quando volta ao positivo. O gráfico marca esse mês com uma linha vermelha tracejada.
- Gráfico "Receitas, despesas e caixa" (`drawFx`): colunas sólidas lado a lado com as cores próprias `--fx-rec` (azul), `--fx-desp` (vinho), `--fx-cx` (verde) e `--fx-neg` (vermelho, caixa negativo), com rótulos de valor quando há espaço (visões Trimestral e Anual). As linhas acumuladas seguem o padrão do comparativo mensal: curva, área preenchida até o zero e pontos.
- Os indicadores ficam todos em uma linha (cartões compactos) a partir de 1300px de largura; abaixo disso, quebram em várias linhas.
- Os planos fixos ficam em `PLANOS`, com percentuais sobre o valor da venda: apartamento À vista, 12x, 36x e 60x; loja À vista e 36x. Vendas com plano "Específico" usam as parcelas da aba **Fluxos específicos**.
- As premissas (início dos recebimentos, despesas de obra, datas de entrega) são editáveis na tela e salvas no `localStorage` (`painel-config`) e no banco compartilhado.

### Visual
- As cores são variáveis CSS em `:root` (`--blue`, `--orange`, `--c3`…`--c6`, `--panel`, `--grid`…), com tema escuro por `prefers-color-scheme` e por `[data-theme="dark"]`. O botão de tema salva a escolha em `painel-tema`.
- **Os gráficos usam só as variáveis, nunca cores fixas.** Para trocar a paleta, altere as variáveis.
- Layout em grade de 3 colunas (`.grid`, `.span2`, `.span3`), que vira 1 coluna abaixo de 1000px.
- **Identidade visual "Análise de Vendas"** (bloco `<style id="layout-azul">` no fim do `<head>`), que sobrepõe o CSS base:
  - azul escuro `--navy: #0B2A5B` como cor principal; `--blue` (barras e série atual) `#123A78`; `--orange` (ano anterior) azul claro `#7FA7DC`;
  - cabeçalho `.hero` com a logo da Egecon, o título em maiúsculas e os botões de ano (`#anoTiles`). Os botões são gerados por um script no fim do `<body>` a partir do `<select id="fAno">`, que continua sendo o filtro real (escondido);
  - indicadores (`.kpi`) em blocos azuis com ícone redondo à esquerda (SVG embutido via `--ico`, que se repete a cada 4 cartões);
  - quadros (`.card`) cinza-claro (`--panel: #EDEFF2`), com borda escura, sombra e título centralizado, dentro de um painel cinza arredondado (`--panel-back`);
  - comparativo mensal com área semitransparente abaixo de cada linha (em `drawLine`);
  - tabelas da aba Corretores e clientes com valores alinhados à esquerda.

## Diretrizes de trabalho

- **Idioma:** toda interface, mensagens e comentários em português do Brasil. Valores em R$ no formato brasileiro (`Intl.NumberFormat("pt-BR")`).
- **Manter o arquivo único:** não introduzir build, frameworks ou arquivos JS/CSS separados sem combinar antes.
- **Estilo de código:** seguir o existente: funções curtas, nomes em português ou abreviados (`emp`, `unid`, `cor`, `cli`), comentários raros e só onde a regra de negócio não é óbvia.
- **Mudanças de visual:** testar primeiro em uma cópia (`teste-layout.html`) e só aplicar ao `index.html` depois de aprovado. Manter a identidade azul escuro e não usar cores fixas nos gráficos.
- **Testar localmente** com um servidor HTTP, porque abrir o arquivo direto (`file://`) impede a leitura das planilhas:
  ```bash
  python -m http.server 8765
  ```
  Depois abra `http://localhost:8765/index.html`. Confira as três abas, os filtros e os temas claro e escuro.
- **Planilhas:** manter os nomes de arquivo e de colunas, porque o painel depende deles. Fechar o Excel antes de fazer commit.
- **Dados sensíveis:** as planilhas têm nomes de clientes e o repositório é público. O `robots.txt` só pede para buscadores não indexarem; ele não protege o acesso.

## Publicar uma atualização

```bash
git add .
```

```bash
git commit -m "Atualiza planilhas"
```

```bash
git push
```

O GitHub Pages republica em cerca de 1 minuto. Identidade do git neste computador: Filipe Andrade · filipeaxavier@hotmail.com.

## Pendências conhecidas
- O aviso "Aba Cadastro de Unidades não foi usada (colunas não reconhecidas ou aba repetida)" aparece ao abrir o painel. Ainda não foi investigado.
- No comparativo mensal, as duas áreas são tons de azul e, onde se sobrepõem, o ano anterior fica pouco visível.
