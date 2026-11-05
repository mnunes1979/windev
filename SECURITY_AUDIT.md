# AUDITORIA COMPLETA DE SEGURANÇA, ARQUITETURA E QUALIDADE — v2026.11

> Como usar: coloca este ficheiro na raiz do repositório e diz ao agente (Claude Code / Codex):
> **"Lê SECURITY_AUDIT.md e executa a auditoria completa. Não alteres código."**

---

## 0. PAPEL E REGRAS DE ATUAÇÃO

Atua como uma equipa sénior combinada: arquiteto de software, engenheiro de segurança aplicacional (AppSec), pentester (white-box), especialista em Supabase/PostgreSQL, especialista em cloud/DevSecOps (Vercel, GitHub), especialista em UX/UI e acessibilidade, e DPO (RGPD).

Data de referência: **5 de novembro de 2026**. Usa como base o OWASP Top 10:2025, OWASP ASVS 5.0, OWASP API Security Top 10, OWASP Top 10 para aplicações LLM, CWE Top 25 e as boas práticas atuais de Supabase, Vercel e GitHub. Quando avaliares versões de dependências, verifica as vulnerabilidades conhecidas **à data atual** (npm audit / OSV / GitHub Advisories), não apenas pela tua memória.

**Regras obrigatórias:**
1. **MODO SÓ LEITURA.** Não alteres, apagues nem faças commit de nada. O único ficheiro que crias é o relatório.
2. **Nada destrutivo em produção.** Testes ao vivo só passivos ou de leitura (ex.: pedir headers, tentar ler uma tabela com a chave pública). Nunca escrever, apagar, fazer brute force, DoS, nem enviar emails/pagamentos reais.
3. **Nunca escrevas segredos completos no relatório.** Mascara sempre: `sk_live_ab12…(mascarado)`, indicando ficheiro e linha.
4. **Prova cada achado** com ficheiro:linha, excerto curto ou comando reproduzível. Sem prova → classifica como "A verificar", nunca como facto.
5. Se não tiveres acesso a algo (painel Supabase, Vercel, GitHub settings), diz exatamente o que não conseguiste verificar e que comando/ecrã o utilizador deve consultar.
6. Pensa como um atacante: para cada achado, descreve o cenário de exploração realista e o impacto (dados expostos, conta comprometida, custo, reputação, RGPD).

---

## FASE 1 — RECONHECIMENTO E IDENTIFICAÇÃO DO PROJETO

Determina e documenta:

- **Origem / quem gerou o código:** procura sinais de Lovable (`lovable-tagger` no `vite.config`, `.lovable/`, commits "lovable-dev[bot]", `src/integrations/supabase/`), Claude Code (`CLAUDE.md`, `.claude/`), Codex (`AGENTS.md`, `.codex/`), Bolt, v0, Cursor ou código manual. Indica o grau de confiança.
- **Stack completa e versões:** framework (React/Vite, Next.js, Remix…), TypeScript (strict ou não), router, state management, UI lib (shadcn, Tailwind…), backend (Supabase, Edge Functions, API routes Vercel, serverless), base de dados, autenticação, storage, pagamentos (Stripe, ifthenpay, etc.), email (Resend, SMTP…), IA (OpenAI, Anthropic, Gemini…), analytics, terceiros.
- **Arquitetura:** diagrama em texto (Mermaid) de cliente → API/Edge → BD/Storage → serviços externos. Identifica fronteiras de confiança.
- **Modelo de dados e tenancy:** single-tenant ou multi-tenant? Como é feito o isolamento (`tenant_id`, `organization_id`, schemas)?
- **Papéis/perfis de utilizador** existentes (anon, user, admin, super-admin, por tenant).
- **Inventário de superfície de ataque:** todas as rotas públicas, rotas autenticadas, rotas admin, endpoints/Edge Functions, webhooks, formulários, uploads, pesquisas, parâmetros de URL, integrações.
- **Dados sensíveis tratados:** dados pessoais (RGPD), dados financeiros, documentos, dados de saúde, credenciais de terceiros.
- **Deploy:** Vercel (prod/preview), domínio(s), variáveis de ambiente esperadas, CI/CD.

---

## FASE 2 — SEGREDOS E CONFIGURAÇÃO

- Procura segredos no código **e no histórico git** (`git log -p`, ou `gitleaks detect` / `trufflehog git file://.` se disponíveis): chaves API, `service_role`/`sb_secret_`, tokens, passwords, chaves privadas, webhooks secrets, connection strings.
- Verifica o que vai para o **bundle do cliente**: tudo com prefixo `VITE_`, `NEXT_PUBLIC_`, `REACT_APP_` é público. Confirma que apenas a chave pública/publishable do Supabase está exposta, nunca a `service_role`/secret.
- Faz `build` (se possível) e procura no `dist/` / `.next/` por segredos, URLs internos, source maps públicos (`*.map` em produção expõe código-fonte).
- `.env*` no `.gitignore`? Existe `.env.example` sem valores reais?
- Chaves de IA/pagamentos chamadas diretamente do frontend? (Crítico: devem passar por backend/Edge Function.)
- Configuração `vercel.json`, `vite.config`, `next.config`, `supabase/config.toml`.

---

## FASE 3 — SUPABASE / BASE DE DADOS (área mais crítica em projetos Lovable)

Assume que **a chave pública do Supabase é conhecida por qualquer atacante** — a única proteção real é RLS + policies + funções bem feitas.

- **RLS ativo em TODAS as tabelas do schema `public`** (e outros schemas expostos pela API). Lista tabelas sem RLS.
- **Qualidade das policies:** procura `USING (true)`, `WITH CHECK (true)`, policies para `anon` em tabelas sensíveis, policies de `UPDATE` sem `WITH CHECK` (permite mudar `user_id`/`tenant_id`/`role`), falta de policies de `DELETE`, policies que confiam em dados que o próprio utilizador controla (ex.: `user_metadata` / `raw_user_meta_data` — editável pelo utilizador; usar `app_metadata` ou tabela de roles).
- **Escalada de privilégios:** o utilizador consegue atualizar a sua própria coluna `role`, `is_admin`, `plan`, `credits`, `tenant_id`?
- **Isolamento multi-tenant:** um utilizador do tenant A consegue ler/escrever dados do tenant B? Testa logicamente cada tabela.
- **Funções `SECURITY DEFINER`:** têm `SET search_path = ''` (ou fixo)? Validam `auth.uid()` e permissões internamente? Estão expostas via RPC a `anon`/`authenticated` sem necessidade? (`REVOKE EXECUTE … FROM anon, authenticated` quando aplicável.)
- **Views:** usam `security_invoker = true`? Views sem isso ignoram RLS.
- **Storage:** buckets públicos que não deviam ser; policies de `storage.objects` por pasta/utilizador; listagem de ficheiros permitida a anon; URLs assinados com expiração adequada.
- **Edge Functions:** `verify_jwt` ativo onde deve; uso de `service_role` apenas no servidor e sempre com verificação do utilizador antes; CORS restrito (não `*` em funções autenticadas); validação de input (zod); tratamento de erros sem fugas de stack traces.
- **Realtime:** canais/tabelas publicados que expõem dados de outros utilizadores?
- **Auth settings** (se acessível via CLI/MCP, senão listar para o utilizador verificar no painel): confirmação de email, proteção contra passwords comprometidas, política de passwords, MFA disponível para admins, Redirect URLs restritos (sem wildcards abertos), rate limits de auth, expiração de JWT/sessões, OAuth providers configurados corretamente, templates de email.
- Se tiveres Supabase CLI ou MCP: corre `supabase db lint` / Security Advisor e Performance Advisor e inclui resultados.
- **Teste ao vivo (só leitura, com a chave pública que está no frontend):** para cada tabela, tenta `select` como `anon`. Reporta qualquer tabela que devolva dados que não deviam ser públicos.
- Índices em falta nas colunas usadas em policies (RLS lento = DoS barato).
- Backups/PITR, migrações versionadas no repo (`supabase/migrations`) vs alterações feitas só no painel.

---

## FASE 4 — AUTENTICAÇÃO, SESSÃO E CONTROLO DE ACESSO

- **Proteção de rotas só no frontend é decorativa.** Verifica que cada operação sensível é autorizada no servidor/BD (RLS, Edge Function), não apenas escondida no React (`if (isAdmin)`). Um atacante altera o JS no browser, o localStorage ou chama a API diretamente.
- IDOR / BOLA: endpoints ou queries que aceitam `id` vindo do cliente sem verificar posse.
- Painéis admin: como se determina que alguém é admin? É verificável/alterável pelo cliente?
- Tokens: onde são guardados (localStorage vs cookies httpOnly), logout invalida sessão, refresh tokens, sessões longas.
- Recuperação de password, magic links, convites: enumeração de utilizadores (mensagens diferentes para email existente/inexistente), expiração, reutilização.
- Brute force: rate limiting em login, registo, reset, OTP, formulários de contacto.
- Open redirects em `?redirect=`, `?next=`, `returnUrl`.
- CSRF em endpoints com cookies.
- Fluxos de pagamento/créditos/planos: o cliente consegue alterar preço, plano ou quantidade? Webhooks de pagamento validam assinatura e idempotência?

---

## FASE 5 — INJEÇÃO E VALIDAÇÃO DE INPUT

- **XSS:** `dangerouslySetInnerHTML`, `innerHTML`, `eval`, `new Function`, `document.write`, renderização de Markdown/HTML de utilizadores sem sanitização (DOMPurify), `href`/`src` com `javascript:` vindo de dados, SVG de utilizadores, conteúdo de IA renderizado como HTML.
- **SQL injection:** queries construídas por concatenação em Edge Functions/API routes, `rpc` com SQL dinâmico (`EXECUTE format(...)` sem `%L`/`%I`), filtros PostgREST montados com strings do utilizador (`.or(\`...${input}\`)`).
- Injeção de comandos, path traversal, SSRF (fetch para URLs fornecidos pelo utilizador, ex.: "importar de URL", previews, webhooks configuráveis).
- Prototype pollution, ReDoS em regex próprias, deserialização insegura.
- Validação de schema em **todo** o input do servidor (zod/valibot), não apenas no formulário do frontend.
- Email header injection em formulários de contacto.

---

## FASE 6 — UPLOAD DE FICHEIROS (ficheiros contaminados)

- Validação de tipo **no servidor** por magic bytes, não por extensão nem pelo `Content-Type` do cliente.
- Lista branca de extensões/MIME; limite de tamanho no servidor e no bucket.
- SVG, HTML, XML aceites? (vetores de XSS). Ficheiros servidos do mesmo domínio da app?
- Ficheiros servidos com `Content-Disposition: attachment` e `X-Content-Type-Options: nosniff` quando apropriado.
- Renomeação (UUID) em vez de nome original; sem path traversal no nome.
- Antivírus/scan (ex.: ClamAV numa Edge Function/worker, ou serviço externo) para documentos de terceiros — sobretudo em plataformas de gestão documental e faturas.
- PDFs/Office com macros, ZIP bombs, imagens com metadados EXIF (fuga de localização/dados pessoais).
- Processamento de ficheiros por IA/OCR: risco de prompt injection embutido no documento.
- Quem consegue ler os ficheiros carregados? (policies de storage por utilizador/tenant.)

---

## FASE 7 — HEADERS HTTP E CONFIGURAÇÃO DE DEPLOY (Vercel)

Faz pedidos reais (`curl -sI https://dominio`) à produção e às páginas principais e avalia:

- `Content-Security-Policy` (idealmente sem `unsafe-inline`/`unsafe-eval` em scripts; `frame-ancestors`, `object-src 'none'`, `base-uri 'self'`, `connect-src` limitado ao Supabase/APIs usadas). Propor CSP concreta para esta app.
- `Strict-Transport-Security` (max-age ≥ 1 ano, includeSubDomains, preload se aplicável).
- `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`, `X-Frame-Options`/`frame-ancestors`, `Cross-Origin-Opener-Policy`.
- Remoção de headers que revelam tecnologia desnecessariamente.
- CORS das APIs/Edge Functions.
- **Preview deployments** da Vercel publicamente acessíveis com dados de produção? Deployment Protection ativa?
- Variáveis de ambiente separadas por ambiente (production/preview/development).
- Rotas de debug, `/api/test`, páginas de admin não linkadas mas acessíveis, `robots.txt`/`sitemap.xml` a expor rotas privadas.
- Firewall/WAF e rate limiting da Vercel configurados?
- Cookies: `Secure`, `HttpOnly`, `SameSite`.
- DNS e email do domínio: SPF, DKIM, DMARC (proteção contra spoofing), CAA, subdomínios órfãos (subdomain takeover).
- TLS e redirecionamento HTTP→HTTPS, `www` vs raiz.

---

## FASE 8 — FRONTEND: O QUE UM ATACANTE CONSEGUE MUDAR NO BROWSER

Parte do princípio de que o atacante tem o código do frontend todo (está no bundle) e pode alterá-lo à vontade.

- Lógica de negócio crítica feita só no cliente (preços, descontos, limites, validação de plano, verificação de permissões, cálculos).
- Dados sensíveis devolvidos pela API e apenas "escondidos" na UI (ex.: query que traz todas as colunas incluindo `email`, `phone`, `notes` internas). Verifica `select('*')`.
- Feature flags/admin desbloqueáveis por localStorage ou query string.
- Dados pessoais em localStorage/sessionStorage, logs de consola em produção, mensagens de erro detalhadas.
- Source maps em produção.
- Scripts de terceiros (analytics, chat widgets, CDNs) sem SRI; risco de supply chain no browser.
- Clickjacking em ações sensíveis.

---

## FASE 9 — FUNCIONALIDADES DE IA (se existirem)

- Chaves de IA nunca no frontend.
- Prompt injection direta e indireta (documentos, emails, páginas web processadas pela IA) — a IA consegue ser levada a revelar dados de outros utilizadores/tenants, o system prompt, ou executar ações (tools) indevidas?
- O contexto enviado ao modelo respeita o isolamento por utilizador/tenant?
- Output da IA renderizado com segurança (sem HTML/JS executável, links validados).
- Limites de uso por utilizador (abuso de custos / "denial of wallet").
- Dados pessoais enviados a fornecedores de IA: base legal RGPD, DPA, região de processamento.

---

## FASE 10 — CADEIA DE FORNECIMENTO E REPOSITÓRIO (GitHub)

- `npm audit` / `pnpm audit` / `bun audit` + `npx osv-scanner` (se disponível). Lista vulnerabilidades críticas/altas com versão corrigida.
- Verifica especificamente versões de React, React DOM, Next.js, Vite, supabase-js e restantes frameworks contra advisories atuais (houve vulnerabilidades críticas recentes em React Server Components / Next.js — confirmar se a versão instalada é afetada).
- Lockfile presente e consistente; dependências abandonadas, typosquatting, pacotes com scripts `postinstall` suspeitos; dependências desnecessárias.
- GitHub (verificar ou listar para o utilizador): repo público vs privado, branch protection em `main`, Dependabot alerts/updates, secret scanning + push protection, 2FA na conta, colaboradores e tokens/deploy keys com acesso.
- GitHub Actions: permissões mínimas (`permissions:`), actions fixadas por SHA, uso perigoso de `pull_request_target`, segredos expostos em logs.
- Integrações com acesso ao repo (Lovable, Vercel, bots) — necessárias e com escopo mínimo?

---

## FASE 11 — RESILIÊNCIA, LOGS E OPERAÇÃO

- Tratamento de erros: falhas devem fechar o acesso (fail-closed), não abri-lo; sem stack traces para o cliente.
- Logging de eventos de segurança (logins falhados, alterações de permissões, ações admin, acessos a documentos) sem registar dados sensíveis.
- Alertas: alguém é notificado de anomalias?
- Rate limiting e proteção contra abuso em formulários públicos (captcha/Turnstile onde faça sentido).
- Backups, plano de recuperação, possibilidade de rodar segredos rapidamente.

---

## FASE 12 — RGPD / PRIVACIDADE (Portugal / UE)

- Política de privacidade e termos, consentimento de cookies real (scripts de tracking só após consentimento).
- Minimização de dados, prazos de retenção, direito ao apagamento (existe forma de apagar conta e dados, incluindo ficheiros?).
- Região dos dados (Supabase/Vercel na UE?), subcontratantes e transferências internacionais.
- Dados pessoais em logs, analytics, emails, ferramentas de IA.

---

## FASE 13 — QUALIDADE DE ARQUITETURA, CÓDIGO, UX/UI E ACESSIBILIDADE

(Menor prioridade que segurança, mas incluir.)
- TypeScript strict, `any` excessivo, código morto, duplicação, componentes gigantes, separação de responsabilidades.
- Tratamento de estados (loading, vazio, erro), feedback ao utilizador, mensagens de erro úteis mas não reveladoras.
- Performance: Lighthouse/Core Web Vitals (se possível), bundle size, imagens, queries N+1.
- Acessibilidade (WCAG 2.2 AA; relevante para o European Accessibility Act): contraste, labels, navegação por teclado, foco, alt text.
- SEO técnico básico em páginas públicas.
- Testes existentes (unitários, e2e) e cobertura das partes críticas.

---

## FORMATO DO RELATÓRIO (OBRIGATÓRIO)

Cria o ficheiro **`SECURITY_AUDIT_REPORT.md`** na raiz com esta estrutura exata, para ser usado depois para gerar o prompt de correção:

```
# Relatório de Auditoria — <nome do projeto>
Data: <data> | Commit auditado: <hash> | Auditor: <Claude Code/Codex + modelo>

## 1. Resumo executivo
- Nota global de segurança (0–10) e nota de qualidade (0–10)
- Nível de risco: Crítico / Alto / Médio / Baixo
- Top 5 riscos em linguagem simples (para cliente/gestão)

## 2. Identificação do projeto
- Origem provável do código (Lovable / Claude Code / Codex / manual) + evidência
- Stack e versões
- Arquitetura (Mermaid)
- Tenancy, papéis, dados sensíveis
- Domínios e ambientes

## 3. Tabela de achados
| ID | Severidade | Categoria (OWASP/CWE) | Título | Ficheiro:linha | Explorável a partir da internet? | Esforço de correção (S/M/L) |

## 4. Achados detalhados
Para cada achado:
### SEC-001 — <título>
- Severidade: Crítica/Alta/Média/Baixa/Info (justificar)
- Categoria: OWASP 2025 Axx / CWE-xxx
- Localização: ficheiro:linha (ou tabela/policy/configuração)
- Evidência: excerto curto ou comando (segredos mascarados)
- Cenário de ataque: passo a passo de como seria explorado
- Impacto: o que o atacante obtém
- Correção recomendada: descrição técnica concreta (+ snippet/SQL proposto)
- Como verificar que ficou corrigido: teste concreto

## 5. Itens "A verificar" (sem acesso ou sem prova suficiente)
Com instruções exatas do que o utilizador deve verificar (painel/comando).

## 6. Pontos positivos (o que já está bem feito)

## 7. Checklist de configuração manual
Supabase Auth, Vercel, GitHub, DNS — o que deve ser ativado nos painéis.

## 8. Plano de remediação priorizado
- Fase 0: imediato (críticos, rotação de segredos)
- Fase 1: esta semana
- Fase 2: este mês
- Fase 3: melhorias contínuas

## 9. Anexo JSON
Bloco ```json com array de achados: id, severity, category, title, location, fix_summary, effort, depends_on.
```

Ao terminar, mostra no chat apenas o resumo executivo e a tabela de achados, e indica o caminho do relatório.
