# DESIGN.md — Contabiliz Design System

> **Versão:** 1.0
> **Última atualização:** 2026-04-03
> **Fonte:** Pesquisa de mercado (NotebookLM), referências Docsumo, Vercel, Rossum, Stampli, Arquivei/Qive, Retool, Metabase + especificações técnicas do projeto.

---

## 1. Princípios de Design

Quatro princípios guiam todas as decisões visuais e de interação do Contabiliz. Quando houver dúvida entre duas abordagens, o princípio é o critério de desempate.

### 1.1 Precisão antes de estética

O produto lida com dados fiscais de validade legal — CNPJ, CPF, chave de acesso de 44 dígitos, valores monetários. O design deve tornar esses dados **inequívocos**. Isso significa alinhamento numérico perfeito em tabelas, hierarquia clara entre campos obrigatórios e opcionais, e estados de erro que explicam o motivo da falha — não apenas sinalizam que algo deu errado.

### 1.2 Status sempre visível

O coração do produto é um pipeline assíncrono. O usuário envia documentos e espera. A ansiedade do estado de espera é resolvida com **disclosure progressivo**: primeiro o resumo do lote ("47 de 50 concluídos"), depois o detalhe por documento. Nenhum processo que dure mais de 10 segundos fica sem um label textual explicando o que está acontecendo.

### 1.3 Densidade com respiro

O público são analistas financeiros em desktop corporativo. Eles preferem **alta densidade** — ver mais linhas sem rolar — mas não querem ser agredidos por informação. A solução é usar compactação nas tabelas (linhas de 36–40px com zebra striping) e respiração nos painéis e dashboards (tabs separando contextos, whitespace generoso entre seções).

### 1.4 Profissional, não burocrático

O produto é sério, mas não antiquado. O modelo mental é a interseção entre a precisão de uma ferramenta de developer experience (Vercel, Linear) e a usabilidade de uma ferramenta fiscal brasileira (Arquivei). Interface limpa, monocromática, com uma única cor de destaque — sem gradientes decorativos, sem ilustrações infantilizadas.

---

## 2. Paleta de Cores

### 2.1 Base neutra

A interface é construída sobre uma escala de neutros de alto contraste, compatível com modo claro e escuro.

| Token | Light mode | Dark mode | Uso |
|---|---|---|---|
| `--color-bg-base` | `#FAFAFA` (Alabaster) | `#0A0A0A` | Fundo da página |
| `--color-bg-surface` | `#FFFFFF` | `#171717` (Cod Gray) | Cards, painéis, modais |
| `--color-bg-subtle` | `#F4F4F5` | `#1C1C1E` | Fundo de tabelas, inputs |
| `--color-bg-muted` | `#E4E4E7` | `#27272A` | Zebra striping, divisores |
| `--color-border` | `#D4D4D8` | `#3F3F46` | Bordas de cards e inputs |
| `--color-border-strong` | `#A1A1AA` | `#52525B` | Bordas de foco, separadores |
| `--color-text-primary` | `#09090B` | `#FAFAFA` | Textos principais |
| `--color-text-secondary` | `#52525B` | `#A1A1AA` | Labels, metadados, captions |
| `--color-text-disabled` | `#A1A1AA` | `#52525B` | Texto inativo |

### 2.2 Cor de destaque (Accent)

Uma única cor de destaque vibrante para ações primárias, links e elementos interativos em foco.

| Token | Valor | Uso |
|---|---|---|
| `--color-accent` | `#0070F3` (Vercel Blue / Blue Ribbon) | Botões primários, links, foco de input, progresso |
| `--color-accent-hover` | `#0060D0` | Estado hover em botões e links |
| `--color-accent-subtle` | `#EFF6FF` / `#0C1A2E` (dark) | Background de badges info, highlights |
| `--color-accent-text` | `#FFFFFF` | Texto sobre fundo accent |

> **Regra:** `--color-accent` aparece em no máximo um elemento por viewport de uma vez. Nunca usar como cor decorativa de fundo de seções ou cards.

### 2.3 Cores de status semânticas

Usadas exclusivamente para comunicar estados de processamento de documentos. **Sempre acompanhadas de label textual** — nunca usar cor isolada para transmitir estado (acessibilidade).

| Token | Light | Dark | Status | Label padrão |
|---|---|---|---|---|
| `--color-status-success` | `#16A34A` | `#4ADE80` | Processado / Confirmado | "Concluído" |
| `--color-status-success-bg` | `#F0FDF4` | `#052E16` | — | — |
| `--color-status-warning` | `#D97706` | `#FCD34D` | Processando / Na fila | "Processando" |
| `--color-status-warning-bg` | `#FFFBEB` | `#1C1400` | — | — |
| `--color-status-error` | `#DC2626` | `#F87171` | Falhou | "Falhou" |
| `--color-status-error-bg` | `#FEF2F2` | `#2D0A0A` | — | — |
| `--color-status-neutral` | `#71717A` | `#A1A1AA` | Pendente / Aguardando | "Aguardando" |
| `--color-status-neutral-bg` | `#F4F4F5` | `#1C1C1E` | — | — |

### 2.4 Modo escuro

O dark mode é **opção primária**, não secundária. A experiência padrão ao criar conta segue a preferência do sistema operacional (`prefers-color-scheme`). O usuário pode fixar o modo no perfil.

O dark mode usa `#0A0A0A` como fundo base (não `#000000` puro — o contraste absoluto cansa em sessões longas) e `#171717` para superfícies elevadas.

---

## 3. Tipografia

### 3.1 Famílias tipográficas

| Família | Uso | Fallback |
|---|---|---|
| **Geist Sans** | Todos os textos de interface: headings, body, labels, botões | `Inter`, `system-ui`, `sans-serif` |
| **Geist Mono** | Tabelas com dados numéricos, chaves de acesso (44 dígitos), CNPJs, CPFs, valores monetários em colunas, blocos de código | `JetBrains Mono`, `Fira Code`, `monospace` |

> **Geist** é a família tipográfica da Vercel, open source e otimizada para interfaces de alta densidade técnica. Combina clareza humanista nos textos gerais com precisão monospace nos dados.

### 3.2 Escala tipográfica

| Token | Tamanho | Peso | Line-height | Uso |
|---|---|---|---|---|
| `--text-xs` | 11px | 400 | 1.5 | Microlabels, badges compactos |
| `--text-sm` | 12px | 400 | 1.5 | Captions, metadados de tabela, timestamps |
| `--text-base` | 14px | 400 | 1.6 | Body text, conteúdo de tabela, labels de form |
| `--text-md` | 15px | 500 | 1.5 | Labels de seção, subtítulos de card |
| `--text-lg` | 18px | 500 | 1.4 | Títulos de card, nomes de batch |
| `--text-xl` | 22px | 600 | 1.3 | Títulos de página |
| `--text-2xl` | 28px | 600 | 1.2 | Números de KPI em dashboards |

> **Regra de pesos:** usar apenas 400 (regular) e 500/600 (medium/semibold). Nunca 700 (bold) em textos de interface — fica pesado contra o fundo claro B2B.

### 3.3 Regra crítica: dados numéricos sempre em Geist Mono

Esta regra é **não-negociável**. Qualquer dado que apareça em coluna de tabela e que o analista precise comparar visualmente linha a linha deve usar `font-family: 'Geist Mono'` com `font-variant-numeric: tabular-nums`.

```css
/* Aplicar em toda célula de tabela com dado numérico ou fiscal */
.table-cell--numeric,
.table-cell--cnpj,
.table-cell--cpf,
.table-cell--chave-acesso,
.table-cell--valor {
  font-family: 'Geist Mono', monospace;
  font-variant-numeric: tabular-nums;
  letter-spacing: -0.01em;
}
```

Fontes proporcionais (Inter, sistema) causam desalinhamento de vírgulas decimais e pontos de milhar em colunas — problema crítico para um analista financeiro identificar valores errados ou inconsistentes.

---

## 4. Iconografia e Elementos Visuais

### 4.1 Ícones

Usar a biblioteca **Lucide Icons** (open source, consistente, tamanhos 16px e 20px). Nenhum ícone personalizado sem aprovação explícita.

| Contexto | Tamanho | Cor padrão |
|---|---|---|
| Ícone dentro de botão | 16px | herda a cor do botão |
| Ícone em tabela (status, tipo de doc) | 16px | `--color-text-secondary` |
| Ícone de ação em toolbar | 20px | `--color-text-secondary` |
| Ícone decorativo em empty state | 40px | `--color-text-disabled` |

**Ícones de tipo de documento:**

| Tipo | Ícone Lucide | Cor |
|---|---|---|
| XML (NF-e) | `FileCode2` | `--color-accent` |
| PDF (DANFE) | `FileText` | `--color-status-error` atenuado |
| Imagem (cupom) | `FileImage` | `--color-status-warning` atenuado |

### 4.2 Status dots

Padrão do Vercel: ponto colorido de 8px acompanhado de label textual. Nunca usar o ponto sozinho.

```
● Concluído     (verde — --color-status-success)
◌ Processando   (âmbar — --color-status-warning, com animação pulse)
● Falhou        (vermelho — --color-status-error)
● Aguardando    (cinza — --color-status-neutral)
● Na fila       (cinza — --color-status-neutral)
```

O estado "Processando" usa `animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite` para indicar que está ativo — único uso de animação em estado de status.

### 4.3 Skeleton screens

Processos que duram mais de 300ms exibem skeleton screens — retângulos arredondados cinza com animação shimmer — em vez de spinners globais. Isso preserva o layout e reduz a percepção de lentidão.

```css
.skeleton {
  background: linear-gradient(
    90deg,
    var(--color-bg-muted) 25%,
    var(--color-bg-subtle) 50%,
    var(--color-bg-muted) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 4px;
}
```

---

## 5. Espaçamento e Layout

### 5.1 Grid e escala de espaçamento

Escala base de 4px. Tokens de espaçamento:

| Token | Valor | Uso típico |
|---|---|---|
| `--space-1` | 4px | Gap entre ícone e label |
| `--space-2` | 8px | Padding interno de badges, gap entre elementos inline |
| `--space-3` | 12px | Padding de células de tabela |
| `--space-4` | 16px | Padding de cards compactos, gap entre campos de form |
| `--space-5` | 20px | Padding horizontal de modais |
| `--space-6` | 24px | Padding de cards padrão, gap entre seções |
| `--space-8` | 32px | Margem entre blocos de conteúdo |
| `--space-10` | 40px | Padding de páginas (lateral) |
| `--space-12` | 48px | Separação entre seções maiores |

### 5.2 Layout de aplicação

```
┌─────────────────────────────────────────────┐
│  Topbar (56px) — logo, nav global, user menu │
├───────────┬─────────────────────────────────┤
│ Sidebar   │  Área de conteúdo principal      │
│ (220px)   │  (max-width: 1280px, centrado)   │
│           │                                  │
│ Nav       │  Padding: 40px lateral           │
│ secundária│  32px entre seções               │
│           │                                  │
└───────────┴─────────────────────────────────┘
```

- **Topbar:** altura 56px, fundo `--color-bg-surface`, border-bottom 1px `--color-border`
- **Sidebar:** largura 220px, colapsável em 1024px ou menos, fundo `--color-bg-surface`
- **Conteúdo:** `max-width: 1280px`, centralizado com `margin: 0 auto`, padding `0 var(--space-10)`

### 5.3 Breakpoints

| Nome | Valor | Comportamento |
|---|---|---|
| `sm` | 640px | — |
| `md` | 768px | Sidebar colapsa para drawer |
| `lg` | 1024px | Layout de duas colunas disponível |
| `xl` | 1280px | Largura máxima do conteúdo |

> **Mobile:** a prioridade de desenvolvimento é desktop (analistas em corporativo). Mobile adapta-se exibindo métricas globais e status de batch — remove upload em massa e tabelas densas.

---

## 6. Componentes de Interface

### 6.1 Botões

| Variante | Uso | Aparência |
|---|---|---|
| `primary` | Ação principal de cada tela (ex: "Enviar batch") | Fundo `--color-accent`, texto branco, border-radius 6px |
| `secondary` | Ações secundárias | Fundo `--color-bg-surface`, borda `--color-border-strong`, texto `--color-text-primary` |
| `ghost` | Ações terciárias, dentro de tabelas | Sem borda, sem fundo, texto `--color-text-secondary`, hover com `--color-bg-subtle` |
| `destructive` | Ações destrutivas (deletar batch) | Fundo `--color-status-error`, texto branco — exige confirmação em modal |

**Tamanhos:**

| Tamanho | Altura | Padding horizontal | Font-size |
|---|---|---|---|
| `sm` | 32px | 12px | 13px |
| `md` (padrão) | 36px | 16px | 14px |
| `lg` | 44px | 20px | 15px |

**Estado de loading:** substitui o label por spinner de 14px + texto "Enviando…". O botão fica `disabled` e `aria-busy="true"`.

### 6.2 Tabelas de dados

As tabelas são o componente central do produto — referência: Retool (densidade) + Arquivei (customização de colunas).

**Anatomia:**

```
┌──────────────────────────────────────────────────────────────┐
│ Toolbar de tabela: busca | filtros | colunas | exportar       │
├──────────────────────────────────────────────────────────────┤
│ Header: label de coluna com ordenação (↑↓)                   │
├──────────────────────────────────────────────────────────────┤
│ Row 1 (bg: --color-bg-surface) ← fundo branco/dark           │
├──────────────────────────────────────────────────────────────┤
│ Row 2 (bg: --color-bg-subtle) ← zebra striping               │
├──────────────────────────────────────────────────────────────┤
│ Row 3 (bg: --color-bg-surface)                               │
├──────────────────────────────────────────────────────────────┤
│ Pagination: ← Anterior | 1 2 3 … 12 | Próxima →             │
└──────────────────────────────────────────────────────────────┘
```

**Especificações:**

- Altura de linha: **36px** (compacto) ou **44px** (padrão) — definido por preferência do usuário
- Zebra striping: linhas pares com `--color-bg-subtle`
- Hover: linha com `--color-bg-muted` + cursor pointer se clicável
- Padding de célula: `0 var(--space-3)` (12px horizontal)
- Header: `--text-sm` (12px), peso 500, `--color-text-secondary`, uppercase com `letter-spacing: 0.05em`
- Dado numérico: `font-family: 'Geist Mono'`, alinhado à direita
- Dado textual: `font-family: 'Geist Sans'`, alinhado à esquerda
- Coluna de status: centralizada, com status dot + label

**Coluna de nome de arquivo:**
- Truncar com `text-overflow: ellipsis` em `max-width: 240px`
- Tooltip com nome completo no hover
- Ícone de tipo de doc à esquerda (16px)

### 6.3 Badges de status

Componente inline para status de documentos e batches.

```
┌──────────────────────┐
│ ● Concluído          │  fundo: --color-status-success-bg
│                      │  texto: --color-status-success
│                      │  font: --text-sm, weight 500
└──────────────────────┘
```

- Border-radius: 9999px (pill)
- Padding: `2px 8px`
- Dot: 6px, margin-right 6px
- Sem ícone — apenas dot + texto

### 6.4 Barras de progresso

**Upload individual (dentro do FileListItem):**

```
[■■■■■■■■□□□□] 67%
```

- `<progress>` HTML nativo, estilizado via CSS
- Altura: 4px, border-radius: 9999px
- Cor preenchida: `--color-accent`
- Cor vazia: `--color-bg-muted`
- Label de porcentagem: `--text-xs`, `--color-text-secondary`, à direita

**Progresso global (batch):**

```
Processando batch · 34 de 50 documentos
[■■■■■■■■■■■■■■□□□□□□□□□□] 68%
```

- Altura: 6px
- Posição: abaixo do header da tela de status
- Texto acima com breakdown: concluídos / na fila / processando / falhos

### 6.5 Área de drag-and-drop (DropZone)

```
┌───────────────────────────────────────────────────────┐
│                                                       │
│         ↑  Arraste os arquivos aqui                   │
│                                                       │
│    ou  [ Selecionar arquivos ]  (botão secondary)     │
│                                                       │
│    XML · PDF · JPEG · PNG · TIFF · WEBP               │
│    Máximo 50 MB por arquivo · até 500 por envio       │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**Estados da DropZone:**

| Estado | Borda | Fundo | Texto |
|---|---|---|---|
| `idle` | 2px dashed `--color-border` | `--color-bg-subtle` | `--color-text-secondary` |
| `drag-over` | 2px solid `--color-accent` | `--color-accent-subtle` | `--color-accent` |
| `error` | 2px solid `--color-status-error` | `--color-status-error-bg` | `--color-status-error` |

- Altura mínima: 160px
- Border-radius: 12px
- Transição de estado: `transition: all 150ms ease`

### 6.6 Cards de sumário (KPI)

Usados na tela de analytics e no topo da tela de status de batch.

```
┌─────────────────────┐
│ Documentos          │  --text-sm, --color-text-secondary
│                     │
│ 1.247               │  --text-2xl, Geist Mono, --color-text-primary
│                     │
│ ↑ 12% este mês      │  --text-xs, --color-status-success
└─────────────────────┘
```

- Fundo: `--color-bg-surface`
- Borda: 1px solid `--color-border`
- Border-radius: 8px
- Padding: `var(--space-5)` (20px)
- Número principal: sempre Geist Mono, `tabular-nums`

---

## 7. Padrões de UX por Tela

### 7.1 PRD-01 — Autenticação

- Tela centralizada, largura máxima 400px
- Logo no topo, formulário no centro, links de suporte no rodapé
- Sem sidebar, sem topbar — tela de shell mínimo
- Campos: `email` e `password` com labels visíveis (não apenas placeholder)
- Botão "Entrar" de largura total (`width: 100%`)
- Feedback de erro inline abaixo do campo relevante, não apenas toast global

### 7.2 PRD-02 — Upload de Batch

**Fluxo de estados e feedback:**

| Estado | Feedback visual | Texto |
|---|---|---|
| `idle` | DropZone com borda dashed | "Arraste os arquivos aqui" |
| `files_selected` | Lista com ícone de tipo + nome + tamanho | Contador: "12 arquivos · 4,3 MB" |
| `presigning` | Botão com spinner + desabilitado | "Preparando upload…" |
| `uploading` | Barra de progresso por arquivo + global | "X de N enviados" |
| `confirming` | Spinner global + skeleton na lista | "Verificando documentos…" |
| `success_full` | Check verde + auto-redirect 2s | "50 documentos enviados com sucesso" |
| `success_partial` | Banner âmbar + lista de falhos | "47 enviados · 3 falharam" |
| `presign_error` | Banner vermelho + botão retry | "Erro ao preparar upload. Tente novamente." |

**Regra de progresso por arquivo:**
- Arquivo `idle`: ícone de tipo + nome + tamanho, sem barra
- Arquivo `uploading`: barra de progresso 4px abaixo do nome
- Arquivo `done`: check verde substituindo o ícone de tipo
- Arquivo `error`: X vermelho + razão resumida em `--text-xs --color-status-error`

**Disclosure progressivo:**
- Por padrão, mostrar apenas arquivos com erro e o contador global
- Botão "Ver todos os arquivos ↓" expande a lista completa
- Máximo de 8 linhas visíveis sem scroll — acima disso, `max-height` com scroll interno

### 7.3 PRD-03 — Status de Batch

**Estrutura da tela:**

```
┌──────────────────────────────────────────────────────────────┐
│  ← Voltar   Batch #a1b2c3   ● Processando   12 min atrás     │
├──────────────────────────────────────────────────────────────┤
│  [KPI: Total] [KPI: Concluídos] [KPI: Processando] [KPI: Falhou] │
├──────────────────────────────────────────────────────────────┤
│  Barra de progresso global                                   │
├──────────────────────────────────────────────────────────────┤
│  Tabs: [Todos (50)] [Concluídos (34)] [Processando (12)] [Falhou (4)] │
├──────────────────────────────────────────────────────────────┤
│  Tabela de documentos (filtrada pela tab ativa)              │
│  Colunas: Arquivo | Tipo | Status | Iniciado | Duração | Razão de erro │
└──────────────────────────────────────────────────────────────┘
```

**Polling:**
- Intervalo adaptativo: 2s se `processing > 0`, parar se batch em estado terminal
- Enquanto polling ativo: indicador discreto "Atualizando…" no canto superior direito
- Sem full-page reload — atualização parcial apenas dos contadores e status

**Linha com status `failed`:**
- Fundo: `--color-status-error-bg` na linha inteira
- Coluna "Razão de erro": texto em `--text-sm --color-status-error`
- Tooltip com razão técnica completa no hover da célula

### 7.4 PRD-04 — Analytics

**Estrutura (padrão Metabase + Retool):**

```
┌──────────────────────────────────────────────────────────────┐
│  Filtros globais: [Período ▼] [Emitente ▼] [Destinatário ▼] │
├──────────────────────────────────────────────────────────────┤
│  Tabs: [Visão Geral] [Fornecedores] [Invoices] [Itens]       │
├──────────────────────────────────────────────────────────────┤
│  Conteúdo da tab ativa                                       │
└──────────────────────────────────────────────────────────────┘
```

**Aba Visão Geral:** 4 KPI cards no topo + mini-gráfico de volume por período abaixo

**Aba Fornecedores:** tabela ranqueada por valor total (Geist Mono na coluna de valor), com CNPJ em coluna separada (Geist Mono) e razão social em Geist Sans

**Aba Invoices:** tabela de notas com colunas: Chave de Acesso (Geist Mono, truncada) | Emitente | Destinatário | Data | Valor Total | Tipo | Confiança

**Aba Itens:** tabela de itens de linha com NCM, CFOP, descrição, quantidade, valor unitário e valor total — todas as colunas numéricas em Geist Mono

**Coluna "Confiança":** relevante para XML (N/A), PDF (barra horizontal 0–100%) e imagem (badge âmbar se < 70%)

---

## 8. Comunicação de Erros

Inspirado em Docsumo (side-by-side review) e Vercel (status dots textuais).

### 8.1 Hierarquia de erros

| Nível | Componente | Onde aparece |
|---|---|---|
| **Erro de documento** | Status dot vermelho + tooltip com razão técnica | Célula de status na tabela |
| **Erro de campo** | Badge vermelho inline na célula | Tabela de documentos com dados extraídos |
| **Erro de batch** | Banner âmbar no topo da tela de status | Quando `partial_failure` |
| **Erro de sistema** | Banner vermelho + botão retry | `presign_error`, `confirm_error` |
| **Erro de rede** | Toast não-bloqueante no canto inferior | Perda temporária de conexão |

### 8.2 Anatomia de mensagem de erro

```
┌─────────────────────────────────────────────────────┐
│ ✕  Falha na extração do documento                   │
│                                                     │
│ nota_fiscal_3.pdf não pôde ser processado.          │
│ Razão: Conteúdo insuficiente para extração de       │
│ texto — documento pode ser um scan de baixa         │
│ resolução.                                          │
│                                                     │
│ [Tentar novamente]  [Ignorar]                       │
└─────────────────────────────────────────────────────┘
```

**Regras:**
- A razão técnica do backend (`error_reason`) nunca é exibida raw — é mapeada para português claro
- Sempre oferecer uma ação (retry, ignorar, contactar suporte)
- Erros de validação local são inline (abaixo do campo) — sem modal
- Erros de sistema são banners ou modais — sem toast para erros críticos

### 8.3 Mapeamento de `error_reason` para mensagem de usuário

| `error_reason` (backend) | Mensagem para o usuário |
|---|---|
| `magic byte mismatch: not an XML file` | "O arquivo não é um XML válido. Verifique se o formato está correto." |
| `XSD validation failed: …` | "A nota fiscal XML não passou na validação do SEFAZ. O arquivo pode estar corrompido ou fora do padrão." |
| `object_not_found` | "O arquivo não chegou ao servidor. Verifique sua conexão e tente novamente." |
| `retries_exhausted: task moved to DLQ` | "O processamento falhou após múltiplas tentativas. Nossa equipe foi notificada." |
| `gemini_terminal: 400/401` | "Erro interno na extração por IA. Por favor, tente novamente mais tarde." |
| `malformed XML: …` | "O arquivo XML está malformado e não pode ser lido." |

---

## 9. Animações e Transições

Animações são **funcionais**, nunca decorativas.

| Elemento | Propriedade | Duração | Easing | Propósito |
|---|---|---|---|---|
| Status dot "Processando" | `opacity` | 2s | `cubic-bezier(0.4,0,0.6,1)` | Indica atividade |
| Skeleton shimmer | `background-position` | 1.5s | `linear` | Percepção de carregamento |
| Expansão de lista (disclosure) | `max-height` | 200ms | `ease-out` | Feedback de abertura |
| Transição de tab | `opacity` | 150ms | `ease` | Orientação de contexto |
| Hover em linha de tabela | `background-color` | 100ms | `ease` | Feedback de interatividade |
| Toast entrada/saída | `transform: translateY` + `opacity` | 200ms | `ease` | Não interrompe o foco |
| Botão press | `transform: scale(0.98)` | 100ms | `ease` | Feedback tátil |

**Proibido:**
- Animações em loop em elementos estáticos (badge "Concluído" não pulsa)
- Transições de página com efeitos elaborados (fade simples ou instantâneo)
- Animações em componentes de dado (tabelas, cards de KPI)
- Respeitar `prefers-reduced-motion: reduce` — desabilitar todas as animações exceto skeleton

---

## 10. Acessibilidade

Requisitos mínimos de acessibilidade para o produto.

### 10.1 Contraste

- Texto normal (< 18px): mínimo **4.5:1** contra o fundo
- Texto grande (≥ 18px ou ≥ 14px bold): mínimo **3:1**
- Componentes interativos e estados de foco: mínimo **3:1**
- A paleta foi definida para atender WCAG AA em ambos os modos

### 10.2 Navegação por teclado

- Todo elemento interativo acessível via `Tab` na ordem lógica do DOM
- `Enter`/`Space` ativam botões e links
- `Escape` fecha modais e dropdowns
- A DropZone de upload é ativável via teclado (`Enter` abre o seletor de arquivo)
- Tabelas suportam navegação por setas (Row navigation)

### 10.3 ARIA obrigatório

| Componente | Atributo |
|---|---|
| Barra de progresso de upload | `role="progressbar"` + `aria-valuenow` + `aria-valuemin` + `aria-valuemax` + `aria-label` |
| Status do batch em atualização | `aria-live="polite"` no container de contadores |
| Mensagem de erro crítico | `role="alert"` |
| Botão em estado de loading | `aria-busy="true"` + `aria-disabled="true"` |
| Arquivo inválido adicionado | `aria-live="polite"` no container da lista |
| Modal | `role="dialog"` + `aria-modal="true"` + `aria-labelledby` |

### 10.4 Foco visível

O outline de foco usa `--color-accent` com offset de 2px. Nunca `outline: none` sem substituto.

```css
:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
  border-radius: 4px;
}
```

---

## 11. Tokens de Design (Referência Rápida)

```css
:root {
  /* Cores — Light Mode */
  --color-bg-base: #FAFAFA;
  --color-bg-surface: #FFFFFF;
  --color-bg-subtle: #F4F4F5;
  --color-bg-muted: #E4E4E7;
  --color-border: #D4D4D8;
  --color-border-strong: #A1A1AA;
  --color-text-primary: #09090B;
  --color-text-secondary: #52525B;
  --color-text-disabled: #A1A1AA;
  --color-accent: #0070F3;
  --color-accent-hover: #0060D0;
  --color-accent-subtle: #EFF6FF;
  --color-status-success: #16A34A;
  --color-status-success-bg: #F0FDF4;
  --color-status-warning: #D97706;
  --color-status-warning-bg: #FFFBEB;
  --color-status-error: #DC2626;
  --color-status-error-bg: #FEF2F2;
  --color-status-neutral: #71717A;
  --color-status-neutral-bg: #F4F4F5;

  /* Tipografia */
  --font-sans: 'Geist Sans', 'Inter', system-ui, sans-serif;
  --font-mono: 'Geist Mono', 'JetBrains Mono', monospace;

  /* Escala de texto */
  --text-xs: 11px;
  --text-sm: 12px;
  --text-base: 14px;
  --text-md: 15px;
  --text-lg: 18px;
  --text-xl: 22px;
  --text-2xl: 28px;

  /* Espaçamento */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 20px;
  --space-6: 24px;
  --space-8: 32px;
  --space-10: 40px;
  --space-12: 48px;

  /* Bordas */
  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 8px;
  --radius-xl: 12px;
  --radius-full: 9999px;
}

/* Dark Mode */
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg-base: #0A0A0A;
    --color-bg-surface: #171717;
    --color-bg-subtle: #1C1C1E;
    --color-bg-muted: #27272A;
    --color-border: #3F3F46;
    --color-border-strong: #52525B;
    --color-text-primary: #FAFAFA;
    --color-text-secondary: #A1A1AA;
    --color-text-disabled: #52525B;
    --color-accent: #3B82F6;
    --color-accent-hover: #60A5FA;
    --color-accent-subtle: #0C1A2E;
    --color-status-success: #4ADE80;
    --color-status-success-bg: #052E16;
    --color-status-warning: #FCD34D;
    --color-status-warning-bg: #1C1400;
    --color-status-error: #F87171;
    --color-status-error-bg: #2D0A0A;
    --color-status-neutral: #A1A1AA;
    --color-status-neutral-bg: #1C1C1E;
  }
}
```

---

## 12. Referências de Mercado Utilizadas

| Empresa | O que foi absorvido |
|---|---|
| **Vercel** | Paleta monocromática de alto contraste, status dots com texto, Geist como família tipográfica, estética DX para pipeline assíncrono |
| **Docsumo** | Side-by-side review, ciclo de vida de documento (Processing → Reviewed → Erred), detalhe de erro por hover |
| **Rossum** | Confidence score visual, fluxo "exception handler", otimização por teclado em revisão |
| **Stampli** | Trilha de auditoria por documento, modelo de comunicação de falha sem "fadiga de notificação" |
| **Arquivei / Qive** | Colunas customizáveis para metadados fiscais brasileiros, referência de conformidade SEFAZ |
| **Retool** | Zebra striping, linhas compactas de 36px, densidade sem sacrificar legibilidade |
| **Metabase** | Tabs para separar contextos em dashboards, evitar "data eyeball attack" com múltiplos cards |
| **Linear** | Modelo de estados de tarefas com disclosure progressivo, label de tempo decorrido |

---

*Documento mantido junto ao código em `invoice-docs/`. Atualizar sempre que uma decisão de design for tomada ou revertida.*
