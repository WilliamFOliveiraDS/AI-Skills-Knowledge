# AI-Skills-Knowledge
Skills desenvolvidas por mim
*PADRONIZAÇÃO NA LOVABLE — O QUE FOI CONFIGURADO*

Pessoal, configurei no nosso workspace da Lovable um Knowledge de segurança e 15 skills. Segue o que é cada coisa e quando entra em ação.

———————————————

*1) KNOWLEDGE DE SEGURANÇA (sempre ativo)*

Esse não depende de gatilho: fica carregado em TODA conversa, mesmo que ninguém peça nada sobre segurança. É a nossa regra base.

O que ele cobre:
• *Arquitetura BFF* — nenhum componente do front chama API externa, banco ou serviço de terceiro direto. Tudo passa pela camada BFF.
• *Segredos* — nada de chave no código ou em .env commitado. Tudo em Secrets.
• *Autenticação* — token em cookie httpOnly, nunca em localStorage. JWT verificado, não só decodificado.
• *Autorização (IDOR)* — estar logado não basta: toda rota confere se o dado é do usuário.
• *Banco* — queries parametrizadas sempre, RLS ativo, retornar só os campos necessários.
• *Upload* — magic bytes, reprocessamento de imagem, SVG bloqueado, limite de 6MB.
• *LGPD* — consentimento, direitos do titular, retenção.
• *Checagem final* — checklist de 30 itens + validar se as APIs externas retornam 200 OK + conferir se nenhuma rota interna ficou quebrada após mudanças.

———————————————

*2) COMO AS SKILLS SÃO ACIONADAS*

Na maioria das vezes é *automático*: a Lovable lê a descrição de todas as skills, compara com o que você pediu e carrega as que fizerem sentido. Não precisa fazer nada.

Mas dá pra *forçar quando quiser*: é só clicar no *+* e selecionar a skill na lista. Aí ela entra com certeza, independente do que a IA achou do seu pedido.

Usem o + quando a skill for essencial pra tarefa — principalmente em pagamento, LGPD e teste de abuso, onde deixar passar custa caro.

———————————————

*3) AS 15 SKILLS*

*seo-onpage*
Quando: criar ou editar página pública, landing page, blog.
Faz: meta tags, Open Graph, schema.org, hierarquia de headings, sitemap e robots.

*acessibilidade*
Quando: qualquer interface — formulário, modal, menu, tabela, botão.
Faz: navegação por teclado, contraste, foco visível, labels para leitor de tela, HTML semântico (WCAG AA).

*lgpd-compliance*
Quando: a funcionalidade coleta, guarda, exibe ou exclui dado pessoal.
Faz: define base legal, minimiza campos coletados, consentimento registrado, direitos do titular, prazo de retenção.

*design-system*
Quando: criar qualquer tela ou componente novo.
Faz: obriga usar tokens de cor, tipografia, espaçamento, raio e breakpoints. É a dona de todo valor visual — as outras skills apontam pra ela.

*mobile-first*
Quando: qualquer tela que o usuário vai abrir no celular.
Faz: constrói da menor tela pra cima, área de toque de 44px, alcance do polegar, tabela virando card, o que fazer sem hover, teclado cobrindo campo.

*integracao-pagamento*
Quando: checkout, assinatura, upgrade de plano, estorno, webhook de gateway.
Faz: assinatura de webhook verificada, idempotência, máquina de estados do pagamento, valor calculado no servidor. Erro aqui custa dinheiro real.

*escrita-cliente*
Quando: escrever texto que o usuário final lê — botão, mensagem de erro, e-mail, tela vazia.
Faz: tom direto e humano, erro que explica o que fazer, sem jargão e sem texto de marketing vazio.

*debug-sistematico*
Quando: algo quebrou, deu erro, parou de funcionar depois de uma mudança.
Faz: obriga reproduzir, isolar a camada e testar hipótese antes de corrigir. Nada de sair chutando solução.

*anti-slop*
Quando: passada final em qualquer código ou texto gerado.
Faz: barra comentário óbvio, nome genérico (data, handleClick, utils.ts), any no TypeScript, componente gigante e código morto.

*explica-pro-cliente*
Quando: precisar reportar entrega, atraso ou problema para cliente leigo.
Faz: traduz o técnico em resultado de negócio, sem sigla e sem enrolação.

*teste-de-abuso*
Quando: funcionalidade com dinheiro, permissão, limite ou fluxo de várias etapas ficou pronta.
Faz: gera os cenários de abuso que checklist genérico não pega — pular etapa, clicar duas vezes ao mesmo tempo, valor negativo, trocar ID, escalar privilégio.

*performance-web*
Quando: página pública, listagem pesada ou tela reclamada como lenta.
Faz: Core Web Vitals, otimização de imagem, tamanho de bundle, re-render desnecessário, paginação.

*formularios*
Quando: cadastro, login, checkout, contato, formulário em etapas.
Faz: validação no servidor também, erro ao lado do campo, bloqueio de duplo envio, máscara e validação real de CPF/CNPJ/CEP/telefone.

*estados-de-tela*
Quando: qualquer tela que exibe dado.
Faz: obriga tratar carregando, vazio, erro, sem permissão, offline, sucesso parcial e muitos dados. Não entregar só o caminho feliz.

*auditoria-de-projeto-existente*
Quando: assumir projeto legado ou código de outra pessoa.
Faz: diagnóstico antes de mexer — onde estão os segredos, como está a autenticação, mapa de rotas, dependências vulneráveis, zonas de risco. Não refatora durante a auditoria.

———————————————

*OBSERVAÇÕES*

• As skills de interface se cruzam de propósito (design-system, mobile-first, acessibilidade, formularios, estados-de-tela, performance-web). Cada uma diz no início o que cobre e pra onde mandar o resto, então não se contradizem.
• Valor concreto (cor, espaçamento, breakpoint) mora só no design-system. As outras referenciam.
• Se perceberem que alguma skill nunca é acionada sozinha, me avisem que ajusto a descrição dela.

Qualquer dúvida, chamem.
