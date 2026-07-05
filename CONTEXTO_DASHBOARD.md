# Contexto — Dashboard de Produção Acumulada

> **Como usar este documento**: cole todo o conteúdo abaixo como a primeira mensagem de uma nova conversa com o Claude, junto com o arquivo `dashboard_producao_acumulada.html` (ou o CSV original, se quiser refazer do zero). Isso evita ter que reexplicar o projeto inteiro.

---

## 1. Sobre o projeto

Dashboard HTML autocontido (funciona offline, sem servidor) para análise de produção de correspondentes bancários, construído a partir do arquivo `TBL_PRODUCAO_ACUMULADA.csv`.

- **Arquivo final**: `dashboard_producao_acumulada.html`
- **Publicado em**: `https://romjsouza-png.github.io/modelo/` (GitHub Pages, repositório público `romjsouza-png/modelo`)
- Todo o dado fica **embutido no próprio HTML** (não lê o CSV em tempo real) — qualquer alteração nos dados exige reprocessar o CSV e gerar um novo HTML.

## 2. Fonte de dados

Arquivo: `TBL_PRODUCAO_ACUMULADA.csv`
- Separador `;`, encoding `utf-8-sig`
- ~142.628 linhas, 52 colunas
- Período coberto pelos dados de exemplo: **02/04/2012 a 30/05/2012**

Colunas usadas no dashboard:
| Coluna | Uso |
|---|---|
| `VALOR_LIBERADO` | valor numérico (formato BR: `1.234,56`), métrica principal |
| `DT_PG` | data de pagamento (`dd/mm/aaaa`), base da série temporal |
| `UF_PRODUCAO` | UF de produção (normalizada para maiúsculo) |
| `COORDENADORAS` | 8 coordenadoras |
| `GERENTES` | 21 gerentes (nomes, incluindo "SEM GERENTE ..." como placeholders) |
| `TIPO_PROPOSTA` | 15 tipos de proposta |
| `CANAL_VENDA` | 2 canais (Código Direto / INSS-Correspondentes) |
| `SITUACAO_PG` | Normal / Devolvido / Reapresentado |
| `Nome_Filial` | ~306 filiais/correspondentes |

Conversão de valores: string BR (`1.234,56`) → float (remover `.` de milhar, trocar `,` por `.`).

## 3. Estrutura de dados embutida no HTML

O HTML contém um objeto JS `DATA` com granularidade **diária por dimensão**, no formato de arrays `[dia, chave, valor, qtd]` para permitir filtragem client-side sem precisar reabrir o CSV:

```js
DATA = {
  meta: { total_valor, total_registros },
  dias: [...],          // lista de todos os dias (YYYY-MM-DD)
  serie: [[dia, valor, qtd], ...],           // total diário geral
  uf: [[dia, uf, valor, qtd], ...],          // top 20 UFs + "OUTROS"
  coord: [[dia, coord, valor, qtd], ...],    // todas as 8 coordenadoras
  tipo: [[dia, tipo, valor, qtd], ...],      // todos os 15 tipos
  canal: [[dia, canal, valor, qtd], ...],    // os 2 canais
  situacao: [[dia, situacao, valor, qtd], ...],
  gerente: [[dia, gerente, valor, qtd], ...],// TODOS os 21 gerentes (sem corte top-N)
  filial: [[dia, filial, valor, qtd], ...],  // top 15 filiais + "OUTROS"
  ger_coord_map: { gerente: coordenadora }   // mapeamento de contexto
}
```

Todas as agregações (filtros de período, rankings, comparativos) são recalculadas **no navegador via JavaScript**, a partir desses arrays — não há chamadas a servidor.

## 4. Estrutura do dashboard (3 abas)

### Aba 1 — Visão Geral
- Filtro de período (Período A) com atalhos: Tudo / Abril-12 / Maio-12
- Comparação com um segundo período (Período B)
- KPIs: valor liberado, contratos, ticket médio, coordenações ativas, taxa devolução/reapresentação (com delta % quando em modo comparação)
- Gráfico de evolução diária (linha)
- Gráfico de valor por UF (top 15, barras horizontais)
- Distribuição por coordenação (donut) + tabela
- Perfil da operação: tipo de proposta, canal de venda, situação de pagamento
- Rankings: top gerentes e top filiais

### Aba 2 — Gerentes & Coordenadoras
- Alternância de dimensão: Coordenadora ou Gerente
- Seleção de "Entidade A" (dropdown) + período próprio
- Comparação com "Entidade B" (outra coordenadora/gerente)
- KPIs por entidade (valor, contratos, ticket médio, % de participação no total)
- Gráfico de evolução diária comparando as entidades selecionadas
- Ranking completo da dimensão escolhida (bar chart), com as entidades selecionadas destacadas em cor diferente
- Tabela comparativa completa

### Aba 3 — Funil por Tipo de Proposta
- Métrica alternável: Valor (R$) ou Quantidade de contratos
- Filtro de período + comparação entre dois períodos
- Visualização em "funil": tipos de proposta ordenados por volume decrescente, com barras proporcionais, % de participação e "queda % vs tipo anterior" no ranking
  - **Nota conceitual**: não é um funil de conversão sequencial real (tipo de proposta é uma categoria, não uma etapa de processo) — é um ranking decrescente visualizado em formato de funil
- Tabela comparativa por tipo de proposta (Período A vs B)

## 5. Especificação visual (design system)

**Estilo**: dashboard moderno, dark mode, paleta em tons de azul, tipografia contemporânea.

```css
/* Cores principais */
--bg: #070C17            /* fundo geral (navy profundo) */
--surface: #101A2E        /* cards/painéis */
--surface-2: #131F36
--border: rgba(137,168,222,0.14)
--text: #EAF1FC
--text-soft: #93A6C6
--text-faint: #5D7195

--accent: #3D8BFF         /* azul primário — Período/Entidade A */
--accent-2: #5EEBD8       /* ciano — Período/Entidade B (comparação) */
--positive: #34D399       /* verde — deltas positivos */
--negative: #FB7185       /* vermelho — devolução/deltas negativos */
--warn: #FBBF66           /* âmbar — reapresentado */
```

**Tipografia** (Google Fonts):
- `Space Grotesk` — títulos e valores de KPI
- `Inter` — texto corrido, nomes em tabelas
- `JetBrains Mono` — labels, datas, valores monetários, eixos de gráfico

**Elementos de identidade visual**:
- Cards em "glass" com borda sutil e linha de destaque em gradiente azul→ciano no topo
- Fundo com glow radial sutil (gradiente azul no canto superior)
- Biblioteca de gráficos: **Chart.js 4.4.1** (via CDN cdnjs)
- Funil construído em CSS puro (barras com `clip-path` trapezoidal), sem biblioteca externa

## 6. Stack técnica

- HTML único, autocontido (CSS + JS inline, sem build step)
- Chart.js carregado via CDN: `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js`
- Fontes via Google Fonts CDN
- Sem dependências de backend, sem frameworks (JS puro/vanilla)
- Compatível com qualquer navegador moderno, funciona 100% offline após carregado (exceto fontes/Chart.js que dependem de CDN)

## 7. Publicação / hospedagem

- Repositório GitHub: `romjsouza-png/modelo` (público)
- Arquivo publicado como `index.html`
- GitHub Pages ativo: Settings → Pages → Source: "Deploy from a branch" → branch `main`, pasta `/ (root)`
- URL pública: **https://romjsouza-png.github.io/modelo/**
- ⚠️ Repositório público = dados visíveis a qualquer pessoa com o link. Se precisar restringir acesso, considerar hospedagem interna (SharePoint, servidor da empresa) em vez de GitHub Pages público.

## 8. Pedidos já implementados (histórico)

1. Dashboard inicial com KPIs, evolução diária, distribuição por UF/coordenação, perfil de operação, rankings
2. Filtros de período (dia/mês) com atalhos + modo de comparação entre dois períodos
3. Redesign visual: dark mode, paleta azul, tipografia moderna (Space Grotesk/Inter/JetBrains Mono)
4. Aba 2: filtros por Gerente/Coordenadora com comparativos entre entidades
5. Aba 3: funil por Tipo de Proposta com métrica alternável e comparativo de períodos
6. Publicação no GitHub Pages

## 9. Como pedir ajustes numa nova conversa

Ao colar este documento, você pode pedir coisas como:
- *"Adicione um filtro por UF na aba 1"*
- *"Quero uma 4ª aba com [algo]"*
- *"Mude a cor de destaque de azul para verde"*
- *"O arquivo CSV foi atualizado, pode reprocessar os dados?"* (nesse caso, anexe o novo CSV)
- *"Publique a nova versão no GitHub"* (nesse caso, vou te dar o HTML atualizado para você repetir o processo de upload no repositório)

Se for só ajuste visual/funcional (sem trocar dados), basta anexar o `dashboard_producao_acumulada.html` atual — não precisa do CSV original.
