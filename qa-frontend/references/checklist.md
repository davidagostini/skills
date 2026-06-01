# Checklist detalhado — QA Front-end (genérico)

Use por tela, por breakpoint e por tema (dark/light). Itens específicos do projeto (rotas, breakpoints exatos, lista de overlays, regras de cor/ícone/asset) vêm do **perfil do projeto** carregado no Passo 0 da skill.

## 1. Console & Rede (faça primeiro)
- [ ] Console sem erros vermelhos (JS) ao carregar cada rota.
- [ ] Sem erro de runtime (`X is not defined`, `Cannot read properties of undefined`).
- [ ] Sem warning de fonte não carregada — sem FOUT/FOIT (texto piscando do fallback).
- [ ] Network sem 404/redirect indevido em assets (imagens, ícones, fontes, mídia).
- [ ] HTML principal servido sem cache agressivo se o projeto exige (evita precisar hard reload).
- [ ] Páginas públicas não disparam chamadas autenticadas indevidas; sem CORS.
- [ ] Sem warnings do framework (keys faltando em listas/carrosséis, etc.).

## 2. Layout & Quebra (por tela)
- [ ] Todas as imagens carregam; sem ícone de imagem quebrada.
- [ ] Ícones seguem a convenção do projeto (ex.: SVG herdando cor; sem emoji se proibido).
- [ ] Texto não vaza do contêiner — testar strings longas (nomes, e-mails, títulos).
- [ ] Sem overflow horizontal em nenhuma largura (do menor mobile à largura máxima).
- [ ] Sem sobreposição: header/nav fixos não cobrem conteúdo; barra inferior não cobre o último item; toast não cobre botões.
- [ ] Tabelas/listas colapsam corretamente no mobile (cards) sem cortar/espremer.
- [ ] Grids respeitam os breakpoints do projeto ao redimensionar lentamente.
- [ ] Alinhamento e espaçamentos consistentes (tokens/escala).
- [ ] Lista vazia → estado vazio explícito; carregando → skeleton/spinner.
- [ ] Cores por token; contraste OK em dark e light; nenhuma cor "presa".
- [ ] Zoom 100/125/150% sem estourar.

## 3. Popups / Modais / Overlays (desktop E mobile)
Para CADA overlay do inventário do projeto (modais, drawers, painéis, toasts):
- [ ] Abre pelo gatilho (desktop e mobile).
- [ ] Fecha por X, **Esc** e clique no **backdrop**.
- [ ] Posicionado; não estoura a viewport no mobile (scroll interno se alto).
- [ ] Scroll-lock: o body atrás não rola.
- [ ] Backdrop cobre tudo e dá contraste.
- [ ] Z-index acima de header fixo, barra inferior e toasts.
- [ ] Foco entra no overlay ao abrir; focus-trap; foco volta ao gatilho ao fechar.
- [ ] Um overlay por vez; sem "fantasma" do overlay após fechar.
- [ ] Drawer anima e fecha ao navegar para um item.
- [ ] Nenhum efeito visual que trave captura/scroll (ex.: blur em sticky/fixed, se o projeto proíbe).

## 4. Navegação por Teclado (Tab)
- [ ] Tab/Shift+Tab alcança todos os botões e links.
- [ ] Ordem de foco lógica (segue a leitura visual).
- [ ] Foco visível (`:focus-visible`) perceptível em dark e light.
- [ ] Sem armadilha de foco fora de modal (carrossel/iframe).
- [ ] Enter/Espaço ativam botões; Enter segue links.
- [ ] Carrosséis: controles focáveis/operáveis; auto-advance pausa no hover/focus.
- [ ] Forms: Tab entra em todos os campos; Enter submete; erro associado ao campo.
- [ ] Elementos clicáveis não-nativos têm role/tabindex/teclado — ou são button/a.
- [ ] Navegação inferior (mobile) alcançável por teclado; item ativo indicado.

## 5. Estados de Interação
- [ ] `:hover` com feedback em todos os interativos (desktop).
- [ ] `:focus-visible` consistente.
- [ ] `:active` não "trava".
- [ ] Loading: submit mostra spinner/disabled; sem duplo submit.
- [ ] Disabled distinto, não focável, cursor not-allowed.
- [ ] Toggle de tema alterna, anima e persiste após reload.
- [ ] Estados de dados (loading/erro/sucesso) cobertos; refetch não "pisca" a tela inteira.
- [ ] Toast aparece, legível, some sozinho e fecha manual.
- [ ] Nada depende só de hover no mobile (touch tem equivalente).

## 6. Fontes & Tipografia
- [ ] Fontes/pesos/itálicos reais (não sintéticos borrados).
- [ ] Ícones alinhados à baseline; tamanho consistente.
- [ ] Sem texto cortado por line-height/overflow apertado.
- [ ] Imagens responsivas sem distorção (aspect-ratio mantido).

## 7. Temas (dark E light) — caça à "cor presa"
> Faça a varredura DUAS vezes: uma em cada tema. Alterne com `preview_resize colorScheme: dark|light` (e/ou o toggle do app).
- [ ] **Nenhum texto branco-no-branco nem preto-no-preto** — todo texto continua legível ao trocar de tema.
- [ ] Texto inverte junto com o fundo (não fica fixo enquanto o fundo muda).
- [ ] Ícones (cor de traço) visíveis nos dois temas — nenhum some no fundo.
- [ ] Bordas, divisórias e `placeholder` continuam perceptíveis nos dois temas.
- [ ] Anel de foco visível nos dois temas (cruza com a frente 4).
- [ ] Nenhum bloco com fundo claro fixo "estourando" no tema escuro (nem o inverso).
- [ ] Contraste ≥ AA medido (4.5:1 texto normal, 3:1 texto grande/ícone) em CADA tema.
- [ ] Sem cor hardcoded (`#fff`/`#000`/`white`/`black`/hex) onde deveria ser token que troca com o tema.
- [ ] Toggle de tema persiste após reload e alterna sem flash.
- [ ] Detecção objetiva: para cada texto, comparar `color` × `background-color` efetivo; sinalizar quase-idênticos ou contraste < AA.

## 8. Acessibilidade
- [ ] Contraste WCAG AA medido (4.5:1 texto, 3:1 ícone/borda) em dark e light.
- [ ] Landmarks (header/nav/main/footer); `alt` em imagens; `aria-label` em botões só-ícone.
- [ ] `aria-live` em toasts e contadores dinâmicos.
- [ ] `prefers-reduced-motion` respeitado (carrosséis/drawers/animações).
- [ ] Zoom de texto 200% sem perda de conteúdo.
- [ ] `lang` correto no `<html>`.

## 9. Robustez de Conteúdo & Rede
- [ ] Conteúdo dinâmico vazio (0 itens) degrada elegante, não quebra.
- [ ] Strings/imagens ausentes têm fallback/placeholder.
- [ ] Caracteres especiais/emoji no conteúdo renderizam.
- [ ] Texto com HTML/`<script>` é escapado (anti-XSS).
- [ ] Offline → mensagem amigável; 401 → redireciona sem loop; 500/timeout → retry, sem skeleton eterno.
- [ ] Listas com 0/1/muitos itens (paginação grande) sem travar scroll.
- [ ] Deep-link/refresh em rota interna funciona (sem 404 do servidor).
- [ ] Voltar/avançar do navegador mantém estado coerente.
- [ ] Rede lenta (3G): observar layout shift (CLS) enquanto carrega.

## 10. Mobile real / Cross-device
- [ ] Touch targets ≥44px.
- [ ] Barra inferior respeita safe-area (notch/gestos).
- [ ] `100vh` não quebra com a barra de endereço (iOS/Safari).
- [ ] Sticky + scroll-lock de modal funcionam no Safari/iOS.
- [ ] Firefox: scrollbar/foco OK.
