# A2AT: App-to-Agent Tunnel
## Uma Proposta de Delegação Efêmera, Escopada e Auditável Entre Aplicações e Agentes de IA Controlados pelo Usuário

**Status deste Documento:** Informativo / Proposta em Rascunho (pré-padronização)
**Categoria:** Experimental
**Autor:** Marcus Duarte — Independente
**Data:** Julho de 2026
**Versão:** 0.1 (rascunho)

---

## Resumo

Este documento propõe o A2AT (App-to-Agent Tunnel), um protocolo de camada de aplicação que permite que aplicações de terceiros solicitem assistência limitada e auditável do próprio agente de IA do usuário — sem que a aplicação precise guardar uma chave de API do provedor do modelo, e sem que o provedor do agente trate a requisição como automação anônima e sem escopo. O A2AT define: (1) um handshake de estabelecimento de sessão efêmero e com sigilo futuro (forward secrecy); (2) um modelo de consentimento escopado e legível por humanos, aplicado na camada do agente e não na camada da aplicação; (3) um log de auditoria à prova de adulteração, coassinado pelas duas partes; (4) atestação da aplicação para impedir phishing de consentimento por clientes falsificados; e (5) mitigações para ataques de "confused deputy" (deputado confuso) e ataques de privacidade por agregação. O A2AT é desenhado explicitamente para ser adotado pelos provedores de IA como uma capacidade nativa — de forma análoga a como o OAuth é implementado pelos provedores de identidade, e não improvisado pelas partes que dependem dele — em vez de ser um workaround construído por aplicações contra os termos de serviço dos provedores.

---

## 1. Introdução e Motivação

### 1.1 O problema

Desenvolvedores independentes que querem adicionar funcionalidades de IA a uma aplicação enfrentam uma escolha desconfortável:

1. **Embutir uma chave de API do provedor** (própria ou uma chave de backend compartilhada). Isso cria um ponto único de falha financeira e de segurança: uma chave vazada pode gerar cobranças de uso ilimitadas antes que a detecção de fraude do provedor intervenha, e o desenvolvedor arca com toda a responsabilidade, independentemente de quem causou o vazamento.
2. **Pedir que cada usuário traga sua própria chave de API** (BYOK — Bring Your Own Key). Isso transfere o risco financeiro para o usuário e atualmente é o padrão mais defensável em produção, mas ainda exige que cada usuário crie de forma independente uma conta de desenvolvedor junto a um provedor de modelo, gere uma chave e gerencie o ciclo de vida dela — uma barreira significativa para usuários não técnicos.
3. **Repasse manual de prompt.** O usuário copia um prompt gerado pela aplicação, cola no seu próprio cliente de chat de IA (ChatGPT, Gemini, Claude etc.), copia a resposta estruturada de volta e cola na aplicação. Isso não exige nenhum gerenciamento de chave, mas é lento, sujeito a erros e não escala para nenhum fluxo que precise de mais de uma rodada de interação.

Nenhuma dessas opções entrega ao usuário aquilo que ele já tem numa assinatura de IA de consumo: um agente de raciocínio já autenticado, já pago e que ele usa e confia diariamente. A lacuna não está na capacidade do modelo — está numa *camada de transporte e confiança* ausente entre uma aplicação arbitrária de terceiros e a sessão de agente já provisionada do usuário.

### 1.2 Por que isso não é simplesmente um "MCP reverso"

O Model Context Protocol (MCP) e o OpenAI Apps SDK formalizam o *agente chamando a aplicação* como uma ferramenta. O A2AT trata a direção inversa — a *aplicação solicitando a ajuda do agente* — mas não pode ser construído como uma reprodução não oficial da sessão de assinatura de um usuário contra os endpoints não documentados de um cliente de chat. Esse padrão já existe de forma informal (tokens OAuth de CLI roteados por proxies locais para reaproveitar a cota de uma assinatura em ferramentas de terceiros) e já foi explicitamente proibido: a Anthropic alterou seus termos em fevereiro de 2026 para não permitir tokens OAuth de assinatura em ferramentas de terceiros, com aplicação de cobrança ativada em abril de 2026, e o Google fez uma mudança equivalente no acesso via Gemini CLI por volta da mesma época. Isso não é uma brecha do ecossistema que o A2AT possa explorar silenciosamente — é um limite deliberado estabelecido pelos provedores, tanto por prevenção de abuso quanto por razões de modelo de negócio (o preço da assinatura pressupõe padrões de uso primário, e o tráfego automatizado de terceiros não se encaixa nesse modelo).

Por isso, o A2AT é delimitado como uma **proposta de padronização**, não um workaround do lado do cliente. Ele define a superfície de protocolo que um provedor precisaria expor para que esse padrão fosse legítimo, auditável e compatível com seus sistemas existentes de cota e prevenção de abuso — de forma comparável a como o "Sign in with Google" exigiu que o próprio Google construísse e operasse o lado do provedor de identidade, e não que aplicações terceiras fizessem engenharia reversa do formulário de login do Google.

---

## 2. Terminologia

- **App**: a aplicação de terceiros que solicita assistência do agente.
- **Agente**: o cliente/runtime de IA do usuário (uma aplicação de chat, um modelo local no dispositivo, ou uma sessão hospedada pelo provedor) atuando sob a autoridade do usuário.
- **Provedor**: a entidade que opera o modelo subjacente do Agente e aplica seus termos de uso (ex.: a empresa por trás do cliente de chat).
- **Principal**: o usuário humano dono tanto da conta no App quanto da sessão do Agente.
- **Sessão**: uma troca A2AT delimitada entre um App e um Agente, com eventos explícitos de abertura/fechamento.
- **Escopo (Scope)**: uma declaração legível por máquina e por humanos do que o App está solicitando e do que o Agente está autorizado a fazer com essa informação.
- Os termos DEVE / DEVERIA / PODE são usados conforme a RFC 2119 para indicar níveis de exigência para uma implementação conforme.

---

## 3. Modelo de Ameaças

O A2AT assume os seguintes adversários e explicitamente **não** assume proteção contra todos eles — cada um é endereçado ou reconhecido como limitação aceita:

| Adversário | Capacidade | Endereçado pelo A2AT? |
|---|---|---|
| Espião de rede (MITM passivo/ativo) | Pode observar ou injetar tráfego entre App e Agente | Sim — chaves efêmeras vinculadas à sessão, proteção contra replay |
| App malicioso/falsificado | Se passa por um App legítimo para phishing de consentimento ou exfiltração de dados | Sim — atestação de App |
| App comprometido (legítimo mas explorado) | App legítimo envia prompts adversariais para exceder o escopo | Parcialmente — mitigações de confused deputy, aplicação de escopo |
| Cliente do Agente malicioso/comprometido | O próprio Agente se comporta mal ou é comprometido no dispositivo | **Não endereçado** — fora do escopo; segurança de endpoint é responsabilidade do Principal e do Provedor |
| O próprio Provedor | O Provedor pode observar todo o tráfego que roteia e pode reter dados conforme suas próprias políticas | **Não endereçado** — o A2AT assume que o Principal já confia no Provedor pelo simples fato de usar seu Agente; o A2AT não tenta esconder dados do Provedor |
| Adversário de agregação | Um único App faz muitas requisições individualmente autorizadas para reconstruir um perfil mais completo do que qualquer concessão isolada implica | Sim — contabilização de consentimento ciente de agregação |

Ser explícito nas duas últimas linhas importa: o A2AT é um protocolo de retransmissão de confiança entre App e Principal, mediado por um Agente em quem o Principal já confia. Não é um protocolo de confidencialidade contra o Provedor, e não pode substituir a segurança em nível de dispositivo.

---

## 4. Visão Geral da Arquitetura

```
 ┌────────┐  1. Requisição de Sessão (assinada, escopada)  ┌───────────────┐
 │  App   │ ───────────────────────────────────────────────▶│    Agente      │
 │        │                                                  │ (confiável ao  │
 │        │◀─────────────────────────────────────────────────│  usuário)      │
 └────────┘  2. UI de consentimento exibida ao Principal      └───────────────┘
      │                                                              │
      │  3. Troca de chave de sessão efêmera (sigilo futuro)         │
      │◀──────────────────────────────────────────────────────────── 
      │
      │  4. Troca de requisição/resposta escopada (uma ou mais rodadas)
      │◀────────────────────────────────────────────────────────────▶│
      │
      │  5. Fechamento de sessão → invalidação de chaves → registro coassinado
      ▼
 ┌─────────────────────────┐
 │ Log à prova de adulteração│  (mantido pelo Agente, visível ao Principal)
 └─────────────────────────┘
```

O Agente — não a API bruta do Provedor — é a fronteira de confiança. O App nunca recebe uma chave de API do provedor de modelo em nenhum momento; ele só se comunica com o Agente, no qual o Principal já confia e que aplica escopo e consentimento em nome do Provedor.

---

## 5. Estabelecimento de Sessão

### 5.1 Troca de chaves efêmera

O A2AT DEVE usar um esquema de acordo de chaves efêmero que forneça sigilo futuro (forward secrecy) — por exemplo, uma troca Diffie–Hellman efêmera em curva elíptica por sessão, seguindo o mesmo princípio de design da família Signal/Noise Protocol. Uma chave de sessão comprometida após o fechamento da sessão NÃO DEVE permitir a descriptografia do tráfego de sessões passadas.

### 5.2 Proteção contra replay e vinculação de chave

Cada token de sessão DEVE ser vinculado ao portador (sender-constrained), e não meramente do tipo bearer. Esse não é um requisito novo — é o mesmo problema que a extensão **DPoP** do OAuth 2.1 (Demonstrating Proof-of-Possession, RFC 9449) já resolve: um token é vinculado a uma chave privada mantida pela parte solicitante, de forma que um token interceptado não pode ser reutilizado por um terceiro que não também possua essa chave. O A2AT DEVERIA reutilizar o DPoP diretamente em vez de definir um novo mecanismo de vinculação.

### 5.3 Fechamento de sessão

No fechamento explícito (iniciado pelo App, pelo Agente, pelo Principal, ou por timeout), todas as chaves de sessão DEVEM ser invalidadas do lado do Agente imediatamente, e quaisquer cópias do lado do App se tornam criptograficamente inúteis devido ao sigilo futuro — não apenas "apagadas por convenção". Um interceptador que capturou o tráfego da sessão não ganha nada ao reproduzi-lo após o fechamento.

---

## 6. Modelo de Consentimento Escopado

### 6.1 O consentimento precisa ser granular, não binário

Um único prompt "Permitir que este app acesse o Agente? Sim/Não" é insuficiente. O A2AT exige declarações de escopo com, no mínimo:

- **Categoria de dado** (ex.: histórico de compras, localização, um documento específico) — não apenas uma justificativa em texto livre.
- **Propósito** (resposta pontual vs. acesso contínuo/recorrente).
- **Retenção** (se o Agente persiste essa troca no seu próprio histórico, e por quanto tempo).

### 6.2 A UI de consentimento precisa resistir a manipulação, não apenas renderizar texto

Como é o App quem fornece o texto de justificativa exibido ao Principal, um App malicioso ou descuidado pode escrever uma justificativa persuasiva mas enganosa — isso é engenharia social direcionada à própria tela de consentimento, distinto de prompt injection direcionado ao modelo. As implementações DEVERIAM estruturar a UI de consentimento de forma que **a categoria de dado e o escopo sejam renderizados pelo Agente a partir de um vocabulário fixo, independente do App** (um conjunto fechado de identificadores de escopo, similar aos escopos do OAuth), enquanto a justificativa em texto livre do App fica visual e semanticamente subordinada a esse rótulo fixo — nunca um substituto para ele.

---

## 7. Design do Log de Auditoria

Todo par de requisição/resposta que trafega pelo túnel DEVE ser registrado em um log que seja:

- **Visível ao Principal** dentro da própria interface do Agente (não a do App).
- **À prova de adulteração**: cada entrada inclui o hash da entrada anterior (encadeamento por hash, similar a um log simplificado estilo Merkle/blockchain), tornando edições retroativas detectáveis.
- **Coassinado**: tanto o App quanto o Agente assinam o registro do que foi solicitado e do que foi divulgado, dando a ambas as partes não repúdio. Sem coassinatura, um cliente de Agente comprometido poderia editar unilateralmente seu próprio log para esconder mau comportamento, e um App poderia posteriormente contestar uma entrada de log legítima.

---

## 8. Atestação da Aplicação

Sem prova de identidade do App, um App clonado ou falsificado poderia abrir uma sessão A2AT alegando ser uma aplicação confiável já autorizada anteriormente, e fazer phishing de consentimento usando um nome familiar. O A2AT exige atestação de App no momento de abertura da sessão, usando mecanismos de plataforma já existentes — Play Integrity API no Android, App Attest no iOS, ou verificação equivalente de assinatura de código — verificados pelo Agente antes de renderizar qualquer UI de consentimento. Isso não é criptografia nova; é a exigência de que o protocolo não pule uma etapa que toda plataforma móvel já fornece exatamente para esse propósito.

---

## 9. Revogação

Dois casos distintos de revogação precisam ser tratados de forma diferente:

1. **Acesso escopado à sessão** — resolvido automaticamente no fechamento da sessão (Seção 5.3).
2. **Concessões permanentes/recorrentes** — o Principal deve poder revogar um escopo recorrente previamente concedido pela UI do Agente a qualquer momento, e a revogação DEVE se propagar antes da próxima requisição sob aquele escopo ser atendida, não apenas "eventualmente". As implementações DEVERIAM tratar concessões permanentes como de curta duração e reconfirmáveis (ex.: solicitar novamente após N dias ou M requisições) em vez de indefinidas.

---

## 10. Mitigações de Confused Deputy

Este é o problema em aberto mais relevante do design. O Agente de um Principal tipicamente tem acesso a ferramentas e fontes de dados além do que qualquer App isolado está autorizado a ver (e-mail, calendário, outros apps conectados via MCP). Um App malicioso ou comprometido pode elaborar uma requisição que, sob o pretexto de um propósito legítimo, induz o Agente a também recorrer a essas *outras* fontes ao compor sua resposta — disfarçando uma solicitação de dado fora de escopo dentro de uma que parece estar dentro do escopo.

O A2AT exige que:

- A aplicação de escopo do Agente DEVE ser avaliada em relação aos **identificadores de escopo declarados** da requisição, não ao conteúdo do prompt em texto livre do App, e DEVE rejeitar ou remover qualquer parte de uma resposta que exigisse acesso fora do escopo declarado — mesmo que o próprio modelo estivesse disposto a produzi-la.
- A aplicação de escopo DEVERIA ocorrer como um filtro rígido na camada de acesso a ferramentas do Agente (ou seja, o modelo simplesmente não tem as outras ferramentas disponíveis durante um turno escopado por A2AT), e não como uma instrução leve pedindo ao modelo para "por favor, ficar dentro do escopo". Instruções em nível de prompt não são uma fronteira de segurança.

Esta seção deveria ser marcada explicitamente como uma área que precisa de análise formal adicional antes que qualquer implementação seja considerada pronta para produção.

---

## 11. Agregação e Limitação de Taxa

Requisições individualmente autorizadas ainda podem ser combinadas ao longo do tempo para reconstruir um perfil mais amplo do que qualquer concessão isolada implica. O A2AT recomenda que Agentes rastreiem a **divulgação cumulativa por escopo, por App, dentro de uma janela deslizante**, e disparem novamente o consentimento quando um limiar for ultrapassado — não apenas limitar por volume de requisições.

---

## 12. Relação com Padrões Existentes

O A2AT deliberadamente não reinventa a roda. Ele compõe:

- **OAuth 2.1** para o enquadramento de autorização.
- **DPoP (RFC 9449)** para tokens vinculados ao portador e resistentes a replay.
- **Escopo de ferramentas ao estilo MCP / Apps SDK** para como o Agente expõe e restringe o acesso a ferramentas durante uma sessão.
- APIs padrão de **atestação de plataforma móvel** para identidade do App.

A contribuição do A2AT não é criptografia nova — é definir a semântica de *consentimento, auditoria e confused deputy* específica da direção App→Agente, que nenhum dos padrões acima trata isoladamente.

### 12.1 Relação com o Agent Client Protocol (ACP)

O Agent Client Protocol (ACP), publicado pela Zed Industries em agosto de 2025, padroniza a comunicação entre um editor de código e um agente de codificação via JSON-RPC, tipicamente iniciado como um subprocesso local. O ACP é estruturalmente próximo do A2AT — também formaliza uma direção "aplicação chama o agente", com ciclo de vida de sessão, streaming e um sistema de permissão para tool calls.

A distinção não é ferramenta-de-desenvolvedor versus interface-para-consumidor; isso é uma *consequência*, não a causa subjacente. A causa subjacente é o **domínio de confiança** que cada protocolo pressupõe:

- O ACP pressupõe um único domínio de confiança: o mesmo usuário escolhe, instala e autentica tanto o editor quanto o agente. Não há terceiro adversarial entre eles, então o ACP não precisa — e não define — chaves de sessão efêmeras vinculadas ao portador com sigilo futuro, atestação de aplicação, consentimento restrito por escopo e resistente a manipulação, ou um log de auditoria à prova de adulteração e coassinado, visível a um usuário final não técnico.
- O A2AT pressupõe dois domínios de confiança independentes e mutuamente desconhecidos: um App de terceiro (um vendor diferente, potencialmente não confiável) solicitando acesso escopado e revogável a um Agente que o Principal já possui e em quem já confia (outro vendor, novamente). Como o App pode ser malicioso ou comprometido, o A2AT exige as primitivas de segurança listadas acima como requisito rígido, não como reforço opcional.

Um teste útil para essa distinção: mesmo um desenvolvedor tecnicamente sofisticado usando um App de terceiro (que ele não escreveu) para acessar sua própria conta de ChatGPT/Gemini/Claude ainda precisaria de atestação e consentimento escopado, porque o App continua sendo um terceiro não confiável independentemente da perícia técnica do Principal. A superfície de autorização simplificada e amigável ao usuário leigo do A2AT é consequência de quem tipicamente usa aplicações de consumo — não é a razão pela qual as primitivas de segurança existem.

Como os dois protocolos endereçam camadas diferentes da mesma direção geral, o A2AT DEVERIA ser posicionado como complementar, e não concorrente: o ACP (ou um transporte de sessão/tool-call similar) poderia plausivelmente servir como camada de transporte subjacente para uma sessão A2AT, com o A2AT contribuindo com a semântica de consentimento entre domínios de confiança, atestação e auditoria que o modelo de ameaças do ACP não exige.

---

## 13. Caminho de Implantação e Adoção pelos Provedores

O A2AT não pode ser implantado unilateralmente por uma aplicação contra um cliente de chat não modificado — isso reproduziria exatamente o padrão que os provedores já se moveram para encerrar (Seção 1.2). Um caminho de adoção realista exige:

1. Uma **implementação de referência e um documento formal** (este documento, iterado em direção a um rascunho mais completo) circulado para feedback de desenvolvedores já atuantes no ecossistema MCP/Apps SDK.
2. Engajamento com a equipe de relações com desenvolvedores ou de padrões de pelo menos um provedor, posicionando o A2AT como uma capacidade complementar ao MCP/Apps SDK (direção inversa), e não como um mecanismo concorrente ou que contorna suas regras.
3. Um piloto estreito e opt-in — por exemplo, um tipo de escopo sancionado pelo provedor dentro de uma superfície existente de Apps SDK/extensão — antes de propor qualquer padronização mais ampla.

---

## 14. Considerações de Segurança (Resumo)

- Sigilo futuro é obrigatório; o comprometimento de uma sessão não deve expor retroativamente o tráfego passado.
- Tokens precisam ser vinculados ao portador (DPoP ou equivalente), não apenas do tipo bearer.
- O consentimento precisa ser rotulado por escopo a partir de um vocabulário fechado, independente do App; o texto de justificativa fornecido pelo App é apenas informativo, nunca autoritativo.
- A aplicação de escopo precisa ser uma fronteira rígida de acesso a ferramentas, não uma instrução de prompt.
- Logs de auditoria precisam ser à prova de adulteração e coassinados pelas duas partes.
- A identidade do App precisa ser atestada por mecanismos de plataforma antes de o consentimento ser solicitado.
- A divulgação cumulativa precisa ser rastreada por escopo para prevenir ataques de agregação.
- O A2AT não protege contra um endpoint de Agente comprometido, um dispositivo comprometido, ou o tratamento de dados do próprio Provedor — isso permanece fora de escopo e deve ser declarado como tal na documentação de qualquer implementação.

---

## 15. Questões em Aberto e Trabalhos Futuros

- Verificação formal da mitigação de confused deputy (Seção 10) usando um framework existente de análise de protocolos (ex.: Tamarin ou ProVerif).
- Vocabulário de escopo padronizado — quem o governa, como é estendido, e como os provedores mantêm consistência entre Agentes.
- Semântica de recuperação quando o Agente está offline ou sofrendo rate-limit pelo seu próprio Provedor no meio de uma sessão.
- Se sessões A2AT deveriam ser observáveis/auditáveis pelo próprio Provedor para detecção de abuso, e o que isso implica para a postura de "não é um protocolo de confidencialidade contra o Provedor" da Seção 3.

---

## Referências

1. IETF RFC 9449 — *OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)*
2. IETF RFC 6749 / OAuth 2.1 (rascunho) — *The OAuth Authorization Framework*
3. Anthropic — Atualização dos Termos de Serviço sobre tokens OAuth de assinatura em ferramentas de terceiros, fevereiro–abril de 2026
4. OpenAI — *Apps SDK: Guide to Authentication and User Consent*
5. Especificação do Model Context Protocol (MCP)
6. Signal Protocol / Noise Protocol Framework — fundamentação de design para troca de chaves efêmera com sigilo futuro

---

*Este é um rascunho (v0.1) destinado a coletar feedback antes de circulação mais ampla. Comentários, críticas e indicações de trabalhos anteriores são bem-vindos.*
