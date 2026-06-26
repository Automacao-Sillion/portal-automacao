# Portal de Automações Sillion — Documentação (Fase 1)

> Registro da etapa concluída: reestruturação do portal, hospedagem no KVM4 e
> consolidação das automações. Atualizado em 26/06/2026.

## 1. Visão geral

Portal interno que centraliza as automações da Sillion. Layout empresarial com
**barra de ícones por setor** (estilo Omie) e **menu lateral de automações**: ao
clicar, o conteúdo abre na mesma tela, com os apps Streamlit embutidos via iframe.

- **URL de produção:** `http://187.77.251.91:85/` (servido pelo KVM4)
- **Vitrine/backup:** `https://automacao-sillion.github.io/portal-automacao/` (GitHub Pages)
- **Repositório:** https://github.com/Automacao-Sillion/portal-automacao

## 2. Estrutura dos arquivos (portal)

| Arquivo | Função |
|---|---|
| `index.html` | Tela inicial: cards dos setores + barra de ícones. Ponto de entrada. |
| `home.html` | Versão de cards usada internamente (legado). |
| `financeiro.html` | Setor Financeiro (índigo) — 5 automações. |
| `fiscal.html` | Setor Fiscal (vermelho) — Renomear arquivos. |
| `faturamento.html` | Setor Faturamento (roxo) — Faturar por OS, Download NFSe/XML. |
| `departamento_pessoal.html` | Setor Departamento Pessoal (âmbar) — Integração Caju. |
| `docker-compose.yml` | Sobe o portal (nginx) na porta 85. |
| `DEPLOY-KVM4.md` | Guia de hospedagem. |
| `README.md` / `.gitignore` | Doc do repo e ignorados. |

Cada setor é uma página própria (modelo multipágina). O clique numa automação
troca o painel inline, sem recarregar; apps externos abrem em `<iframe>`.

## 3. Onde fica cada coisa

**Editar/versionar (máquina local):**
`C:\Users\sitra\OneDrive - SITRACK SERVIÇOS TECNOLOGICOS LTDA\Área de Trabalho\Projetos_Padronizado\autom_portal_automacao`
> O banco do Git fica FORA do OneDrive em `C:\gitdirs\portal-automacao.git`
> (o `.git` da pasta é só um ponteiro). Isso evita o OneDrive corromper o repositório.

**Produção (KVM4 — IP 187.77.251.91):**
- `/root/portal-automacao/` → arquivos do portal (servidos pelo container `sillion-portal`, porta 85).
- `/opt/portal-apps/` → app consolidado das automações (container `sillion-apps`, porta 8500).
- `/opt/app`, `/opt/detalhamento`, `/opt/fatomie`, `/opt/caju` → código-fonte original das automações (backup).

## 4. Containers em produção (KVM4)

| Container | Porta | O que é |
|---|---|---|
| `sillion-portal` | 85 | nginx servindo o portal HTML |
| `sillion-apps` | 8500 | **1 container** com as 4 automações Streamlit, roteadas por caminho |
| `home-filebrowser-1` | 8080 | filebrowser (pré-existente, separado) |

Rotas das automações no `sillion-apps` (porta 8500):
- `/baixas/` — Baixas Recebimento
- `/download/` — Download de NFSe/XML
- `/faturar/` — Faturar por OS
- `/caju/` — Integração Caju

> Antes eram 4 containers (8501–8504), um por automação. Foram consolidados em
> **um só** (`sillion-apps`) com nginx + supervisor internos, sem alterar o código
> dos apps. Os containers e imagens antigos foram removidos.

## 5. Quais apps cada setor embute

- **Financeiro:** Cartões e Baixas → **nuvem** (`*.streamlit.app`, com `?embed=true`).
- **Faturamento:** Faturar por OS → `:8500/faturar/`; Download → `:8500/download/` (KVM4).
- **Departamento Pessoal:** Caju → `:8500/caju/` (KVM4).
- Lançamentos e Fluxo de Caixa (Financeiro) e Renomear (Fiscal) são botões de ação
  que disparam webhook (`WEBHOOK_BASE` no topo do `<script>` de cada arquivo).

## 6. Fluxo de atualização

```
1. Editar os arquivos na máquina local
2. git add -A && git commit -m "..." && git push          (vai para o GitHub)
3. No KVM4:  cd /root/portal-automacao && git pull          (atualiza o site)
```
Para reconstruir o app de automações (raro): no KVM4,
`cd /opt/portal-apps && docker compose up -d --build`.

## 7. Decisões e aprendizados importantes

- **Git fora do OneDrive:** o OneDrive corrompia/apagava o `.git`. Solução: banco do
  git em `C:\gitdirs\` (a pasta só guarda um ponteiro). OneDrive pode ficar pausado em operações git.
- **Mixed content:** página HTTPS (GitHub Pages) não embute iframe HTTP. Por isso o
  portal "de verdade" roda no KVM4 em HTTP, onde os apps HTTP embutam sem bloqueio.
- **Streamlit na nuvem:** precisa de `?embed=true` para embutir (sem ele, dá loop de
  redirecionamento). Apps self-hosted (KVM4) embutam sem `embed` e rolam normalmente.
- **Encoding:** manter os arquivos em **UTF-8** (acentos). Evitar comandos de
  reencode em lote no PowerShell (quebraram acentos e truncaram arquivos antes).
- **Multiusuário:** cada usuário tem sessão Streamlit isolada; uso simultâneo é ok.

## 8. Pendências / próximas etapas

- [ ] **Página de consulta ao banco de dados** (listar/filtrar registros via API própria no KVM4).
- [ ] Setor **Contabilidade** (hoje "em breve").
- [ ] Opcional: domínio + HTTPS no KVM4 (tirar o `:85` da URL).
- [ ] Opcional: auto-altura dos iframes self-hosted (encaixe perfeito).
