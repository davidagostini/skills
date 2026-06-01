# Checklist detalhado — QA Backend / API (genérico)

Use por módulo/endpoint, com request real contra o ambiente subido. Itens específicos (como subir, seed, contrato exato, regras de soft delete/auditoria) vêm do **perfil do projeto** carregado no Passo 0 da skill.

## 0. Ambiente & smoke test (obrigatório, primeiro)
- [ ] App sobe sem erro (comando do projeto); health OK.
- [ ] Migrations aplicaram (sem pendência).
- [ ] Login com a credencial de seed → sucesso + token/cookie.
- [ ] Endpoint "quem sou eu" → identidade resolvida pela claim canônica.
- [ ] Endpoint crítico do módulo → sucesso com contrato correto.
- [ ] Suíte de testes do projeto passa.

## 1. Auth & Sessão
- [ ] Login inválido → 401, mensagem sem revelar se é usuário ou senha.
- [ ] Token expira no TTL configurado; expirado → 401.
- [ ] Cookie HttpOnly + SameSite + Secure (em HTTPS).
- [ ] Refresh/renovação funciona (se houver).
- [ ] Logout invalida sessão; token velho não reentra.
- [ ] Identidade lida SEMPRE da claim canônica — nunca do body/query do cliente.
- [ ] Código one-time / link único é single-use e expira.

## 2. Contrato de resposta
- [ ] Listas seguem o contrato padrão do projeto (ex.: `{items,total,page,pageSize}`).
- [ ] Erro tem contrato consistente (código + mensagem), não stack cru.
- [ ] Paginação respeita page/pageSize; limites (page 0, pageSize gigante) tratados.
- [ ] Filtros e ordenação validam campos (sort em campo inexistente não 500).
- [ ] Datas/timezone consistentes (UTC/ISO).

## 3. Status HTTP
- [ ] 200/201/204 no sucesso conforme verbo.
- [ ] 400 validação.
- [ ] 401 sem auth · 403 sem permissão · 404 inexistente/soft-deleted.
- [ ] 409 duplicidade.
- [ ] 500 não expõe stack/segredos/connection string.

## 4. Autorização / IDOR
- [ ] Endpoint exige a permissão correta.
- [ ] Usuário sem permissão → 403 (não 200 vazio nem 500).
- [ ] Trocar o id na URL não acessa recurso de outro usuário/tenant (IDOR).
- [ ] Não dá pra escalar o próprio perfil/permissão sem direito.

## 5. Persistência & Migrations
- [ ] Migrations up e down rodam limpas.
- [ ] Sem FK-sombra (shadow FK) por convenção indevida.
- [ ] Todo campo do payload realmente persiste (bug clássico: campo no DTO que não grava).
- [ ] Índices/unique onde necessário (chaves naturais, email).
- [ ] Relacionamentos M2M gravam e leem corretos.
- [ ] Concorrência: update simultâneo não corrompe (rowversion/optimistic se aplicável).

## 6. Validação de entrada
- [ ] Validação no backend independente do front.
- [ ] Obrigatórios, tamanhos, formatos (email, telefone, etc.).
- [ ] Duplicidade → 409, não 500.
- [ ] Payload extra/desconhecido tratado de forma consistente.
- [ ] Injeção (SQL/HTML) não quebra nem executa.

## 7. Soft delete (se o projeto exige)
- [ ] Exclusão marca status + timestamp; registro permanece no banco.
- [ ] Item deletado some das listas e de GET por id (404).
- [ ] Nenhum DELETE físico em endpoint.
- [ ] Update não "ressuscita" registro deletado.
- [ ] Unique constraint considera o soft delete (recriar com mesma chave após exclusão).

## 8. Auditoria (se exigida)
- [ ] Toda alteração gera evento com diff antigo→novo.
- [ ] Evento registra autor (claim), timestamp, ação.
- [ ] Trilha consultável e filtrável.
- [ ] Operações sensíveis (login, mudança de permissão) auditadas.

## 9. Jobs & Logs
- [ ] Jobs em background enfileiram e processam.
- [ ] Falha de job tem retry/registro, não some silencioso.
- [ ] Logs estruturados com correlação por request.
- [ ] Logs NÃO vazam senha, token, dados sensíveis.

## 10. Ambiente & Operação
- [ ] Health/version responde.
- [ ] Modo manutenção bloqueia conforme esperado (se houver).
- [ ] Parâmetros gerais (TTL, políticas) lidos de config, não hardcoded.
- [ ] Storage/anexos: upload, download, limite de tamanho/tipo (se houver).
