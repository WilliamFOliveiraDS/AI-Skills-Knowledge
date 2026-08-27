# REGRAS BASE DO PROJETO — ARQUITETURA E SEGURANÇA

Esta é uma instrução permanente de projeto. Ela deve ser aplicada em TODA
tela, componente, endpoint ou funcionalidade criada a partir de agora,
mesmo que o pedido do usuário não mencione arquitetura ou segurança
explicitamente. Se uma implementação violar qualquer regra abaixo, ALERTE
antes de prosseguir e sugira a versão correta.

---

## PARTE 1 — ARQUITETURA: BACKEND-FOR-FRONTEND (BFF)

### 1.1 Camada BFF obrigatória
- Nenhum componente de frontend pode chamar diretamente APIs externas,
  bancos de dados ou serviços de terceiros.
- Toda comunicação com serviços externos (APIs REST, GraphQL, banco de
  dados, autenticação, pagamentos, etc.) deve passar por uma camada BFF
  (edge functions / API routes / server functions do projeto).
- O frontend só deve conversar com endpoints internos do próprio BFF
  (ex: `/api/bff/...`).

### 1.2 Responsabilidades do BFF
- Agregar dados de múltiplas fontes/serviços quando necessário, entregando
  ao frontend já no formato pronto para exibição.
- Transformar, filtrar e validar dados ANTES de enviar ao cliente — o
  frontend não deve tratar dados brutos de API externa.
- Esconder detalhes de implementação de serviços externos (URLs, headers,
  formatos de payload) do frontend.
- Tratar erros de forma padronizada e devolver mensagens amigáveis ao
  cliente, nunca stack traces ou erros crus da API externa.

### 1.3 Estrutura de pastas
- Organizar os endpoints do BFF em pasta clara, ex: `/api/bff/[recurso]`.
- Cada domínio/funcionalidade deve ter seu próprio módulo BFF
  (ex: `/api/bff/usuarios`, `/api/bff/pedidos`, `/api/bff/pagamentos`).
- Separar claramente: camada de UI (`components/`) x camada BFF
  (`api/` ou `server/`) x integrações externas (`services/` ou `clients/`).

### 1.4 Contratos de dados
- Definir tipos/interfaces claros para as respostas do BFF, desacoplados
  do formato original da API externa.
- O frontend deve conhecer apenas o "contrato" do BFF, nunca o schema
  da API externa.

### 1.5 Fluxo padrão para novas funcionalidades
Sempre que for pedida uma nova tela ou funcionalidade que dependa de dados,
seguir este fluxo por padrão, sem que precise ser pedido explicitamente:
1. Criar/atualizar o endpoint BFF correspondente.
2. O BFF chama o(s) serviço(s) externo(s) necessário(s).
3. O BFF formata e retorna os dados prontos.
4. O componente de frontend consome apenas o endpoint BFF.

---

## PARTE 2 — SEGURANÇA

### 2.1 Gestão de chaves e segredos
- Nenhuma API key, token ou segredo pode ser escrito diretamente no código
  ou em arquivos `.env` commitados.
- Todo segredo deve ficar em Cloud > Secrets (AI Studio) e ser referenciado
  no código apenas por nome, nunca pelo valor.
  - Errado: `const stripe = new Stripe('sk_live_ABCD1234...')`
  - Certo: `const stripe = new Stripe(secret.stripe.api.key)`
- A `service_role key` do Supabase (ou equivalente admin do Firebase) NUNCA
  pode aparecer no front-end em hipótese alguma — só no servidor/BFF,
  salva como Secret.

### 2.2 Headers de segurança e deploy
- Implementar Content-Security-Policy (CSP), X-Content-Type-Options e
  Strict-Transport-Security (HSTS) em todas as respostas HTTP.
- Suprimir headers que revelam a tecnologia usada (ex: `X-Powered-By`).
- Usar uma lib como "helmet" para ativar várias proteções de header de
  uma vez.
- Desligar modo debug em produção.
- Desligar geração de source maps na build de produção.
- Bloquear acesso à pasta `.git` no servidor.

### 2.3 Proteção contra CSRF
- Toda requisição que altera estado (POST, PUT, DELETE, PATCH) precisa de
  proteção anti-CSRF: token CSRF ou atributo `SameSite` configurado nos
  cookies de sessão.

### 2.4 Rate limiting
- Implementar rate limiting em endpoints sensíveis ou que consomem muitos
  recursos, para evitar brute-force e DoS. Restringir por IP.

### 2.5 Autenticação (cookies, não localStorage)
- Tokens de autenticação NUNCA devem ser salvos em `localStorage`.
- Salvar o token em cookie com as flags:
  ```js
  res.cookie('auth_token', token, {
    httpOnly: true,  // JavaScript da página não consegue ler
    secure: true,    // só trafega por HTTPS
    sameSite: 'Lax', // ajuda a barrar ataques cross-site
  })
  ```
- Todo JWT recebido deve ser VERIFICADO com a chave secreta — nunca apenas
  decodificado sem checar assinatura/validade.

### 2.6 Autorização (além da autenticação)
- Estar logado não é suficiente. Toda rota que retorna dados de um usuário
  específico precisa checar se o dado pertence a quem está pedindo (ou se
  quem pede é admin). Não confiar em nenhum ID vindo do cliente sem validar
  a posse (padrão conhecido como **IDOR — Insecure Direct Object
  Reference**).
  - Errado:
    ```js
    app.get('/api/dados', requireAuth, (req, res) => {
      const dados = db.query('SELECT * FROM dados_sensiveis')
      res.json(dados) // devolve os dados de todos os usuários!
    })
    ```
  - Certo:
    ```js
    app.get('/api/users/:id/dados', requireAuth, (req, res) => {
      if (req.user.id !== req.params.id && req.user.role !== 'admin')
        return res.status(403).json({ error: 'Sem permissão' })
      const dados = db.query('SELECT * FROM dados WHERE user_id=$1', [req.params.id])
      res.json(dados)
    })
    ```
- Atenção especial a Server Actions (Next.js): parecem seguras por
  rodarem no servidor, mas são endpoints públicos como qualquer outro.
  Cada uma precisa checar autenticação E autorização internamente.

### 2.7 Banco de dados
- **SQL Injection:** usar SEMPRE consultas parametrizadas / prepared
  statements. Nunca concatenar texto do usuário na query.
  - Certo: `db.query('SELECT * FROM users WHERE login = $1', [login])`
  - Se usar Prisma, evitar `$queryRawUnsafe`. Validar campos permitidos
    com Zod para prevenir Operator Injection.
- Nunca retornar a linha inteira do banco: selecionar apenas os campos
  necessários (menor privilégio). Ex: perfil retorna nome/foto, não a
  linha completa com senha hash, IDs de pagamento, notas internas.
- **Mass Assignment:** nunca gravar direto no banco o payload recebido do
  usuário sem filtrar. Usar lista branca de campos permitidos (ex: nome,
  bio, foto) — nunca aceitar campos como `"role"` vindos do cliente.

### 2.8 Firebase/Supabase — RLS e isolamento de dados
- Ativar Row Level Security (RLS) em todas as tabelas com dados sensíveis.
- Um usuário nunca deve conseguir consultar dados de outro usuário, mesmo
  manipulando requisições via DevTools/Burp Suite (ex: trocar um ID de
  usuário na URL/payload para acessar dados de outra conta).
- Validar autorização tanto no client quanto (principalmente) no
  backend/RLS.

### 2.9 Upload de arquivos
- Nunca confiar no nome ou content-type declarado do arquivo — ler os
  magic bytes reais (ex: libs "file-type" em JS ou "python-magic" em
  Python) para verificar o tipo verdadeiro.
- Nunca salvar a imagem como recebida: reprocessar com uma lib como Sharp
  para remover metadados e recodificar do zero, eliminando payloads
  escondidos.
  ```js
  const limpa = await sharp(arquivoRecebido)
    .rotate()
    .withMetadata(false)
    .jpeg({ quality: 85 })
    .toBuffer()
  ```
- BLOQUEAR upload de arquivos `.SVG` — podem carregar JavaScript embutido
  e viram vetor de XSS armazenado. Não devem ser aceitos em nenhuma
  hipótese.
- Nunca manter o nome original do arquivo enviado pelo usuário: renomear
  no servidor após os processos acima.
- Limitar tamanho de upload a 6MB por arquivo.

### 2.10 Cookies e consentimento (LGPD)
- Exibir banner de cookies para qualquer cookie não estritamente
  necessário, pedindo consentimento ANTES de ativá-lo — com opção real de
  recusar (não apenas um botão "OK").
- Registrar e armazenar o consentimento do usuário (não repúdio, exigido
  pela LGPD).

### 2.11 LGPD e privacidade por design
- Coletar apenas os dados pessoais estritamente necessários.
- Implementar mecanismos para o titular: acessar seus dados, corrigir
  dados errados, excluir dados, saber com quem foram compartilhados, e
  revogar consentimento a qualquer momento.
- Manter registro de tratamento de dados (o que é coletado, por quê, por
  quanto tempo).
- Dados sensíveis devem ser criptografados em trânsito (HTTPS) e em
  repouso (no banco).
- Ter um plano de resposta a incidentes (a LGPD exige comunicação com a
  ANPD e os titulares afetados em caso de vazamento).

### 2.12 Checklist de validação (teste do F12)
Antes de considerar qualquer funcionalidade pronta, verificar na aba Rede
(Network) do navegador se a resposta da API retorna:
- Senhas ou hashes
- IDs de pagamento/Stripe
- Campos internos/notas administrativas
- Dados de outros usuários

Se qualquer um desses aparecer, é um vazamento de campos e deve ser
corrigido antes do deploy.

### 2.13 Nomenclatura não previsível de endpoints (obscuridade como camada extra)
- Os endpoints do BFF NÃO devem seguir padrões óbvios/previsíveis de REST
  como `api/v1/auth-login`, `api/v1/auth-logout`, `api/v1/users`,
  `api/v1/admin`. Esses nomes são os primeiros que qualquer scanner,
  wordlist de brute-force (ex: dirbuster, ffuf) ou atacante tenta.
- Preferir nomes que não revelem a função do endpoint por si só. Exemplos
  de abordagem:
  - Usar segmentos com termos neutros/genéricos em vez do nome real do
    recurso (ex: em vez de `/api/v1/admin`, algo como `/api/v1/ops-x7k`).
  - Não usar palavras-gatilho óbvias como `admin`, `login`, `auth`,
    `delete`, `internal`, `debug`, `test` nos paths públicos.
  - Considerar adicionar um segmento/token aleatório e estável no path de
    rotas sensíveis (ex: um slug gerado uma vez e reaproveitado), em vez
    de nomes descritivos diretos.
  - Evitar expor a estrutura interna do sistema pelo nome das rotas (ex:
    não nomear endpoints com nomes de tabelas do banco).
- IMPORTANTE — isso é OBSCURIDADE, não segurança de verdade:
  - Nomes de endpoint SEMPRE ficam visíveis para quem inspeciona o
    tráfego de rede (aba Network/F12) ou o bundle do frontend, então essa
    técnica nunca deve ser tratada como proteção suficiente.
  - Toda rota, independentemente do nome, continua exigindo autenticação
    e autorização completas (ver 2.5 e 2.6) — nome incomum não dispensa
    checar dono do dado, verificar JWT, RLS, etc.
  - O objetivo aqui é apenas dificultar reconhecimento automatizado e
    enumeração por bots/scanners, reduzindo ruído de ataques oportunistas
    — não é uma barreira contra um atacante que já está de olho no
    tráfego real da aplicação.

---

## PARTE 3 — SEGURANÇA ADICIONAL (APLICAÇÃO E OPERAÇÃO)

### 3.1 Validação de entrada em TODO endpoint
- Todo endpoint do BFF deve validar as entradas com um schema (Zod ou
  equivalente) antes de qualquer processamento: `body`, `query` e `params`.
- Rejeitar (400) tudo que não bater com o schema. Nunca "tentar entender"
  entrada malformada.
- O schema é também a lista branca (ver 2.7 — Mass Assignment): campos não
  declarados no schema são descartados, nunca repassados ao banco.
- Validar tipos, formatos, tamanhos máximos de string, faixas numéricas e
  valores enumerados permitidos.

### 3.2 XSS e sanitização de saída
- Nunca usar `dangerouslySetInnerHTML` (React) ou equivalente com conteúdo
  vindo do usuário sem sanitizar antes.
- Se for necessário renderizar HTML enviado por usuário, sanitizar com
  DOMPurify (ou lib equivalente) no servidor antes de salvar E antes de
  renderizar.
- Escapar dados do usuário em qualquer template, atributo HTML ou URL.
- Nunca injetar dados do usuário dentro de `<script>`, `eval()`,
  `new Function()` ou handlers inline.
- CSP (ver 2.2) é a rede de proteção, não a defesa principal.

### 3.3 SSRF (Server-Side Request Forgery)
- Se o BFF precisar buscar uma URL fornecida pelo usuário (importar imagem
  por link, preview de URL, webhook configurável, integração customizada),
  isso é um vetor crítico de SSRF.
- Regras obrigatórias nesse caso:
  - Usar allowlist explícita de domínios permitidos. Nunca aceitar
    qualquer URL.
  - Bloquear IPs internos/privados e loopback (127.0.0.1, ::1, 10.x.x.x,
    172.16-31.x.x, 192.168.x.x, 169.254.169.254 — metadata de cloud).
  - Bloquear redirecionamentos para destinos não permitidos (validar o
    destino final, não só a URL inicial).
  - Aceitar apenas os schemes `http`/`https` — nunca `file://`, `gopher://`,
    `ftp://`, etc.
  - Definir timeout curto e limite de tamanho de resposta.

### 3.4 Senhas, sessão e recuperação de conta
- Hash de senha SEMPRE com bcrypt ou argon2. Nunca MD5, SHA1 ou SHA256
  puro.
- Token de reset de senha: uso único, expiração curta (ex: 15–30 min),
  aleatório e criptograficamente seguro, invalidado após o uso.
- Ao trocar a senha, invalidar TODAS as sessões ativas do usuário.
- Não revelar se um e-mail existe no sistema (user enumeration): mensagens
  de login e de "esqueci a senha" devem ser genéricas
  (ex: "Se este e-mail estiver cadastrado, enviaremos as instruções").
- Aplicar rate limiting específico e mais rígido em login, reset de senha
  e cadastro (ver 2.4).
- Definir expiração de sessão e renovação de token.

### 3.5 CORS restritivo
- Nunca usar `Access-Control-Allow-Origin: *`, principalmente em conjunto
  com credenciais/cookies.
- Usar allowlist explícita de origens permitidas, definida por ambiente.
- Permitir apenas os métodos e headers realmente necessários.

### 3.6 Webhooks e callbacks de terceiros
- Todo webhook recebido (Stripe, gateways de pagamento, etc.) DEVE ter sua
  assinatura verificada com o secret do provedor antes de qualquer ação.
  Sem isso, qualquer pessoa pode simular um evento de "pagamento aprovado".
- Usar o corpo bruto (raw body) para validar a assinatura, não o JSON já
  parseado.
- Tratar webhooks como idempotentes: o mesmo evento pode chegar mais de
  uma vez e não pode duplicar efeitos (cobrança, liberação de acesso).
- Validar que o evento pertence à conta/ambiente correto.

### 3.7 Logs e auditoria
- Registrar ações sensíveis com quem/o quê/quando: login, falha de login,
  troca de senha, mudança de permissão/role, exclusão de registros,
  operações financeiras, acesso a dados sensíveis.
- NUNCA logar: senhas, tokens, JWTs, chaves de API, dados de cartão,
  payload completo de requisições com dados pessoais.
- Logs de erro em produção não devem expor stack trace ao cliente (ver 1.2)
  — apenas ID de erro correlacionável no servidor.
- Definir prazo de retenção dos logs, compatível com LGPD (ver 2.11).

### 3.8 Dependências e supply chain
- Manter o lockfile (`package-lock.json` / `yarn.lock`) commitado.
- Rodar `npm audit` (ou equivalente) periodicamente e antes de deploy;
  tratar vulnerabilidades altas/críticas antes de publicar.
- Não adicionar dependências desnecessárias ou sem manutenção ativa.
- Habilitar alertas automáticos de vulnerabilidade no repositório.

### 3.9 Ambientes separados
- Manter dev, staging e produção isolados, com bancos e segredos
  diferentes em cada ambiente.
- NUNCA usar dados reais de clientes em dev/staging — usar dados
  anonimizados ou sintéticos (também é exigência de LGPD).
- Chaves de teste em dev, chaves de produção só em produção.

### 3.10 Rotação de segredos e menor privilégio nas chaves
- Toda chave deve ter o menor escopo possível (ex: chave restrita do
  Stripe em vez de chave total; usuário de banco com permissões mínimas).
- Ter procedimento definido de rotação de segredos, executado
  imediatamente em caso de suspeita de vazamento.
- Revogar chaves de ex-integrações e de pessoas que saíram do time.

### 3.11 Backup testado
- Ter backup automático e periódico do banco de dados.
- A restauração precisa ser TESTADA periodicamente — backup nunca testado
  não é backup.
- Backups devem ser criptografados e ter retenção definida.
- É a principal defesa contra ransomware e perda acidental de dados.

### 3.12 Anti-bot em formulários públicos
- Aplicar captcha/Turnstile (ou equivalente) em cadastro, login,
  recuperação de senha e formulários de contato.
- Complementa — não substitui — o rate limiting (ver 2.4).

---

## PARTE 4 — CHECAGEM FINAL OBRIGATÓRIA

Ao final de QUALQUER implementação, alteração ou refatoração, antes de
declarar a tarefa concluída, executar e REPORTAR a checagem abaixo,
item por item. Não basta dizer "está seguro" — é necessário validar cada
ponto individualmente e informar o status de cada um
(OK / NÃO SE APLICA / PENDENTE + o que falta).

### 4.1 Checklist de segurança (validar um a um)

| # | Item | Status |
|---|---|---|
| 1 | Nenhuma chave/segredo hardcoded no código ou em .env commitado | |
| 2 | Segredos referenciados via Secrets, nunca por valor | |
| 3 | service_role key / chave admin ausente do front-end | |
| 4 | Nenhum fetch direto a API externa no componente de frontend (BFF respeitado) | |
| 5 | Headers de segurança ativos (CSP, HSTS, X-Content-Type-Options) | |
| 6 | Header X-Powered-By e afins suprimidos | |
| 7 | Proteção CSRF em rotas que alteram estado | |
| 8 | Rate limiting em endpoints sensíveis e de login | |
| 9 | Token em cookie httpOnly + secure + sameSite (não em localStorage) | |
| 10 | JWT verificado com chave secreta (não apenas decodificado) | |
| 11 | Autorização checada por rota (dado pertence ao usuário / é admin) — sem IDOR | |
| 12 | Server Actions (Next.js) com auth + authz internos | |
| 13 | Consultas parametrizadas em 100% das queries (sem concatenação) | |
| 14 | Resposta retorna apenas campos necessários (sem hash, IDs internos, PII extra) | |
| 15 | Lista branca de campos na escrita (sem Mass Assignment / role vindo do cliente) | |
| 16 | RLS ativo nas tabelas sensíveis | |
| 17 | Upload: magic bytes validados, imagem reprocessada, SVG bloqueado, arquivo renomeado, limite de 6MB | |
| 18 | Banner de cookies com opção real de recusa + consentimento registrado | |
| 19 | Direitos do titular (LGPD) implementados | |
| 20 | Endpoints com nomenclatura não previsível | |
| 21 | Validação de schema (Zod) em body/query/params de todo endpoint | |
| 22 | Sem dangerouslySetInnerHTML sem sanitização / saída escapada (XSS) | |
| 23 | Proteção SSRF em qualquer fetch de URL fornecida pelo usuário | |
| 24 | Senha com bcrypt/argon2; reset de uso único; sessões invalidadas na troca | |
| 25 | Sem user enumeration em login/reset | |
| 26 | CORS com allowlist explícita (sem `*` com credenciais) | |
| 27 | Assinatura de webhook verificada (raw body) + idempotência | |
| 28 | Logs de auditoria presentes e sem dados sensíveis | |
| 29 | Dependências auditadas, sem vulnerabilidade crítica pendente | |
| 30 | Debug desligado, source maps desligados, .git bloqueado (deploy) | |

### 4.2 Validação de conexões com APIs externas
- Listar todas as integrações externas usadas pela funcionalidade
  implementada/alterada.
- Verificar se cada chamada a API externa está retornando **200 OK**
  (ou o status de sucesso esperado para aquele endpoint).
- Para cada integração, reportar: nome do serviço, endpoint chamado,
  status retornado.
- Se alguma retornar erro (4xx/5xx), NÃO declarar a tarefa concluída —
  apontar o erro, a causa provável (credencial ausente, URL errada,
  payload inválido, permissão) e corrigir.
- Verificar também se o tratamento de erro está implementado para o caso
  de a API externa ficar indisponível (timeout, fallback, mensagem
  amigável — ver 1.2).

### 4.3 Integridade das rotas internas (dependências entre endpoints)
Este é um ponto CRÍTICO e deve ser checado sempre que uma rota for criada,
renomeada, movida ou removida.

**Contexto:** endpoints do sistema consomem uns aos outros. Exemplo: a rota
`/api/financeiro-dashboard` busca dados de `/api/financeiro-table`. Se
`/api/financeiro-table` for renomeada ou removida sem atualizar quem a
consome, o dashboard para de funcionar silenciosamente, porque continua
apontando para uma rota que não existe mais.

**Procedimento obrigatório a cada alteração de rota:**
1. **Mapear consumidores:** antes de renomear/remover qualquer rota,
   buscar em todo o projeto todas as referências àquele path (componentes,
   hooks, services, outros endpoints do BFF, testes, configs).
2. **Atualizar todas as referências:** nenhuma referência ao path antigo
   pode permanecer no código.
3. **Verificar o inverso:** conferir se todas as rotas referenciadas no
   código realmente existem. Qualquer referência a um endpoint inexistente
   é um erro a ser reportado.
4. **Testar o fluxo ponta a ponta:** validar que a tela/funcionalidade que
   depende da rota alterada continua carregando os dados corretamente.
5. **Reportar o mapa de dependências** ao final, no formato:
   ```
   Rota alterada: /api/[antiga] → /api/[nova]
   Consumidores encontrados e atualizados:
     - components/FinanceiroDashboard.tsx (linha X)
     - api/bff/[outro-endpoint] (linha Y)
   Referências órfãs restantes: nenhuma
   Fluxo testado: OK
   ```
6. **Preferir não quebrar contratos:** quando possível, evitar renomear
   rotas já em uso. Se for necessário, considerar manter a rota antiga
   temporariamente redirecionando para a nova, até que todos os
   consumidores estejam migrados.

### 4.4 Formato do relatório final
Ao concluir qualquer tarefa, apresentar:
1. O que foi implementado/alterado.
2. Tabela da checklist 4.1 preenchida (com status de cada item).
3. Status das integrações externas (4.2).
4. Mapa de dependências de rotas, se alguma rota foi tocada (4.3).
5. Pendências e riscos conhecidos, se houver.

Se algum item estiver PENDENTE, dizer isso claramente — nunca marcar como
OK aquilo que não foi de fato verificado.

---

## RESUMO DE COMPORTAMENTO ESPERADO DA IA

- Sempre que gerar código que lida com dados, seguir o fluxo BFF (Parte 1)
  por padrão.
- Sempre que gerar código de autenticação, upload, banco de dados ou
  endpoints, aplicar as regras de segurança (Parte 2) automaticamente,
  sem que precise ser solicitado.
- Ao criar qualquer novo endpoint no BFF, usar nomenclatura não previsível
  (2.13) desde a primeira geração, sem esperar ser pedido — e nunca tratar
  isso como substituto de autenticação/autorização reais.
- Aplicar também as regras da Parte 3 por padrão: validação com schema em
  todo endpoint, sanitização contra XSS, proteção SSRF, hash correto de
  senha, CORS restritivo, verificação de assinatura de webhook e logs sem
  dados sensíveis.
- Se identificar uma violação em código já existente (chave hardcoded,
  fetch direto a API externa em componente, endpoint sem checar dono do
  dado, upload aceitando SVG, etc.), avisar explicitamente antes de
  prosseguir e propor a correção.
- SEMPRE executar a checagem final da Parte 4 ao concluir qualquer tarefa:
  checklist item a item, status das APIs externas (200 OK) e verificação
  de integridade das rotas internas e seus consumidores. Nunca declarar
  uma tarefa concluída sem esse relatório.
