# A2AT: App-to-Agent Tunnel
## Uma Proposta de Delegação Efêmera, Escopada e Auditável entre Aplicações e Agentes de IA Controlados pelo Usuário

**Status do Documento:** Proposta de Rascunho / Informativo (Pré-padronização)  
**Categoria:** Experimental  
**Data:** Julho de 2026  
**Versão:** 0.1 (rascunho)  
**Autor:** Marcus Duarte — Independente  

---

## Resumo

Este documento propõe o **A2AT (App-to-Agent Tunnel)**, um protocolo de camada de aplicação para que aplicações de terceiros solicitem assistência delimitada e auditável do agente de IA do próprio usuário — sem que a aplicação precise reter chaves de API do provedor do modelo e sem que o provedor trate a requisição como automação anônima e sem escopo. 

O A2AT define:
1. Um handshake de estabelecimento de sessão efêmero e com segurança de encaminhamento (*forward secrecy*).
2. Um modelo de consentimento legível por humanos e restrito por escopo, aplicado na camada do agente e não da aplicação.
3. Um registro de auditoria inviolável e assinado por ambas as partes (*co-signed*).
4. Atestação da aplicação para mitigar ataques de *phishing* de consentimento por clientes forjados.
5. Mitigações para ataques de deputado confuso (*confused-deputy*) e baseados em agregação de privacidade.

O A2AT foi explicitamente projetado para ser adotado por provedores de IA como uma capacidade nativa — analogamente a como o OAuth é implementado por provedores de identidade — e não como uma solução de contorno construída por aplicações contra os termos de serviço dos provedores.

---

## 1. Introdução e Motivação

### 1.1 O Problema
Desenvolvedores independentes que desejam adicionar recursos de IA a uma aplicação enfrentam uma escolha desconfortável:

1. **Embutir uma chave de API do provedor** (seja própria ou de um backend compartilhado). Isso cria um ponto único de falha financeira e de segurança: uma chave vazada pode gerar cobranças de uso ilimitadas antes que a detecção de fraude do provedor intervenha, e o desenvolvedor assume total responsabilidade, independentemente de quem causou o vazamento.
2. **Solicitar que cada usuário traga sua própria chave de API** (BYOK — *Bring Your Own Key*). Isso transfere o risco financeiro para o usuário e é atualmente o padrão de produção mais defensável, mas ainda exige que cada usuário crie de forma independente uma conta de desenvolvedor em um provedor de modelos, gere uma chave e gerencie seu ciclo de vida — uma barreira significativa para usuários não técnicos.
3. **Retransmissão manual de comandos (prompts).** O usuário copia um comando gerado pela aplicação, cola-o em seu próprio cliente de chat de IA (ChatGPT, Gemini, Claude, etc.), copia a resposta estruturada de volta e a cola na aplicação. Isso não requer nenhum gerenciamento de chaves, mas é lento, propenso a erros e não escala para fluxos de trabalho que exigem mais de uma iteração de ida e volta.

Nenhuma dessas opções oferece ao usuário o que ele de fato possui em uma assinatura de IA de consumo: um agente de raciocínio já autenticado e pago, no qual ele confia e utiliza diariamente. A lacuna não reside na capacidade do modelo, mas sim na ausência de uma *camada de transporte e confiança* entre uma aplicação de terceiros arbitrária e a sessão do agente já provisionada do usuário.

### 1.2 Por que isto não é simplesmente um "MCP reverso"
O *Model Context Protocol* (MCP) e o SDK de Apps da OpenAI formalizam o *agente chamando a aplicação* como uma ferramenta. O A2AT aborda a direção inversa — a *aplicação solicitando a ajuda do agente* —, mas não deve ser construído como uma reexecução não oficial da sessão de assinatura do usuário contra os endpoints não documentados de um cliente de chat. 

Esse padrão já existe informalmente (tokens OAuth de CLI roteados através de proxies locais para reutilizar a cota de uma assinatura em ferramentas de terceiros) e já foi explicitamente proibido: a Anthropic alterou seus termos em fevereiro de 2026 para proibir tokens OAuth de assinatura em ferramentas de terceiros, com a aplicação do faturamento ativada em abril de 2026, e a Google realizou uma mudança equivalente no acesso à CLI do Gemini no mesmo período. Isso não é um descuido no ecossistema que o A2AT possa explorar silenciosamente — é um limite deliberado definido pelos provedores tanto por razões de prevenção de abusos quanto de modelo de negócios (a precificação por assinatura pressupõe padrões de uso em primeira pessoa; o tráfego automatizado de terceiros não se enquadra nesse modelo).

O A2AT é, portanto, delimitado como uma **proposta de padronização**, e não como uma solução alternativa do lado do cliente. Ele define a superfície de protocolo que um provedor precisaria expor para que esse padrão seja legítimo, auditável e compatível com seus sistemas de cota e prevenção de abusos existentes — comparável a como o "Sign in with Google" exigiu que a Google construísse e operasse o lado do provedor de identidade, e não meramente que as partes confiantes fizessem engenharia reversa no formulário de login da Google.

---

## 2. Terminologia

- **App (Aplicação):** a aplicação de terceiros que solicita assistência ao agente.
- **Agent (Agente):** o cliente/tempo de execução de IA do usuário (um aplicativo de chat, um modelo local no dispositivo ou uma sessão hospedada pelo provedor) agindo sob a autoridade do usuário.
- **Provider (Provedor):** a entidade que opera o modelo subjacente do Agente e aplica seus termos de uso (por exemplo, a empresa por trás do cliente de chat).
- **Principal (Titular):** o usuário humano que possui tanto a conta no App quanto a sessão do Agente.
- **Session (Sessão):** uma única troca delimitada do A2AT entre um App e um Agente, balizada por eventos explícitos de abertura e fechamento.
- **Scope (Escopo):** uma declaração legível por máquina e por humanos sobre quais dados o App está solicitando e o que o Agente está autorizado a fazer com eles.
- Os termos **DEVE (MUST)**, **DEVERIA (SHOULD)** e **PODE (MAY)** são utilizados conforme a especificação RFC 2119 para indicar os níveis de exigência para uma implementação em conformidade.

---

## 3. Modelo de Ameaças

O A2AT assume os seguintes adversários e explicitamente **não** pressupõe proteção contra todos eles — cada um é abordado ou reconhecido como uma limitação aceita:

| Adversário | Capacidade | Abordado pelo A2AT? |
| :--- | :--- | :--- |
| **Interceptador de rede (MITM passivo/ativo)** | Pode observar ou injetar tráfego entre o App e o Agente. | **Sim** — chaves efêmeras vinculadas à sessão, proteção contra repetição. |
| **App malicioso/forjado** | Imita um App legítimo para realizar phishing de consentimento ou exfiltrar dados. | **Sim** — atestação do App. |
| **App comprometido (legítimo, mas explorado)** | App legítimo envia comandos adversariais para ultrapassar o escopo. | **Parcialmente** — mitigações de deputado confuso, imposição de escopo. |
| **Cliente Agente malicioso/comprometido** | O próprio Agente se comporta mal ou está comprometido no dispositivo. | **Não abordado** — fora de escopo; a segurança do endpoint é responsabilidade do Titular e do Provedor. |
| **O próprio Provedor** | O Provedor pode observar todo o tráfego que roteia e pode reter dados conforme suas próprias políticas. | **Não abordado** — o A2AT assume que o Titular já confia no Provedor por virtude de usar seu Agente; o A2AT não tenta ocultar dados do Provedor. |
| **Adversário por agregação** | Um único App faz muitas requisições autorizadas individualmente para reconstruir um perfil mais amplo do que qualquer concessão única permitiria. | **Sim** — contabilização de consentimento consciente de agregação. |

Ser explícito sobre as duas últimas linhas é fundamental: o A2AT é um protocolo de retransmissão de confiança entre o App e o Titular, mediado por um Agente no qual o Titular já confia. Não se trata de um protocolo de confidencialidade contra o Provedor e não pode substituir a segurança ao nível do dispositivo.

---

## 4. Visão Geral da Arquitetura

```text
 ┌────────┐   1. Requisição de Sessão (assinada, escopada)  ┌───────────────┐
 │  App   │ ───────────────────────────────────────────────▶│    Agente     │
 │        │                                                 │ (confiável)   │
 │        │◀────────────────────────────────────────────────│               │
 └────────┘   2. Interface de Consentimento para o Titular  └───────────────┘
      │                                                             │
      │  3. Troca de chaves de sessão efêmeras (forward-secure)     │
      │◀────────────────────────────────────────────────────────────
      │
      │  4. Troca de requisição/resposta escopada (uma ou mais iterações)
      │◀───────────────────────────────────────────────────────────▶│
      │
      │  5. Fechamento da sessão → invalidação de chaves → registro co-assinado
      ▼
 ┌─────────────────────────┐
 │  Log inviolável         │  (mantido pelo Agente, visível ao Titular)
 └─────────────────────────┘

```

O Agente — e não a API bruta do Provedor — constitui a fronteira de confiança. O App nunca recebe uma chave de API do provedor de modelos em momento algum; ele apenas se comunica com o Agente, no qual o Titular já confia e que impõe o escopo e o consentimento em nome do Provedor.

---

## 5. Estabelecimento de Sessão

### 5.1 Troca de chaves efêmeras

O A2AT **DEVE** utilizar um esquema de acordo de chaves efêmeras que forneça segurança de encaminhamento (*forward secrecy*) — por exemplo, uma troca efêmera de *Elliptic-Curve Diffie–Hellman* por sessão, seguindo o mesmo princípio de design da família de protocolos Signal/Noise. Uma chave de sessão comprometida após o encerramento da sessão **NÃO DEVE** permitir a decifragem do tráfego de sessões passadas.

### 5.2 Proteção contra repetição e vinculação de chaves

Cada token de sessão **DEVE** ser restrito ao remetente (*sender-constrained*) e não apenas ao portador. Este não é um requisito inédito — trata-se do mesmo problema que a extensão **DPoP** (*Demonstrating Proof-of-Possession*, RFC 9449) do OAuth 2.1 já resolve: um token é vinculado a uma chave privada mantida pela parte requisitante, de modo que um token interceptado não pode ser reutilizado por terceiros que não possuam também essa chave. O A2AT **DEVERIA** reutilizar o DPoP diretamente em vez de definir um novo mecanismo de vinculação.

### 5.3 Fechamento de sessão

No fechamento explícito (iniciado pelo App, pelo Agente, pelo Titular ou por estouro de tempo/*timeout*), todas as chaves de sessão **DEVEM** ser invalidadas imediatamente no lado do Agente, e quaisquer cópias no lado do App tornam-se criptograficamente inúteis devido à segurança de encaminhamento — não sendo apenas "excluídas por convenção". Um interceptador que tenha capturado o tráfego da sessão não obtém nenhuma vantagem ao reexecutá-lo após o encerramento.

---

## 6. Modelo de Consentimento Escopado

### 6.1 O consentimento deve ser granular, não binário

Uma simples solicitação do tipo "Permitir que este app acesse o Agente? S/N" é insuficiente. O A2AT exige declarações de escopo contendo, no mínimo:

* **Categoria de dados** (por exemplo, histórico de compras, localização, um documento específico) — e não apenas uma justificativa em texto livre.
* **Finalidade** (resposta única versus acesso contínuo/recorrente).
* **Retenção** (se o Agente mantém essa troca em seu próprio histórico e por quanto tempo).

### 6.2 A interface de consentimento deve resistir à manipulação

Como o App fornece o texto de justificativa exibido ao Titular, um App malicioso ou descuidado pode redigir uma justificativa persuasiva, porém enganosa — o que configura engenharia social direcionada à própria tela de consentimento, diferindo da injeção de comandos direcionada ao modelo.

As implementações **DEVERIAN** estruturar a interface de consentimento de modo que a **categoria de dados e o escopo sejam renderizados pelo Agente a partir de um vocabulário fixo e independente do App** (um conjunto fechado de identificadores de escopo, semelhante aos escopos do OAuth), enquanto a justificativa em texto livre do App permaneça visualmente e semanticamente subordinada a esse rótulo fixo — nunca o substituindo.

---

## 7. Design do Registro de Auditoria (Audit Log)

Cada par de requisição/resposta que transita pelo túnel **DEVE** ser registrado em um log que seja:

* **Visível ao Titular** dentro da própria interface do Agente (não na interface do App).
* **Inviolável (*tamper-evident*):** cada entrada inclui o hash da entrada anterior (encadeamento de hashes, como em um log simplificado no estilo Merkle/blockchain), de modo que edições retroativas sejam detectáveis.
* **Co-assinado:** tanto o App quanto o Agente assinam o registro do que foi solicitado e do que foi divulgado, garantindo o não repúdio para ambos os lados. Sem a co-assinatura, um cliente Agente comprometido poderia editar unilateralmente seu próprio log para ocultar comportamentos inadequados, e um App poderia posteriormente contestar uma entrada legítima do log.

---

## 8. Atestação da Aplicação

Sem a prova de identidade do App, um App clonado ou forjado poderia abrir uma sessão A2AT alegando ser uma aplicação confiável e previamente autorizada, realizando phishing de consentimento sob um nome familiar.

O A2AT exige a atestação do App no momento da abertura da sessão utilizando mecanismos de plataforma existentes — *Android Play Integrity API*, *iOS App Attest* ou verificação equivalente de assinatura de código —, checada pelo Agente antes de renderizar qualquer interface de consentimento. Isto não representa uma criptografia nova; é a exigência de que o protocolo não pule uma etapa que toda plataforma móvel já fornece exatamente para essa finalidade.

---

## 9. Revogação

Dois casos distintos de revogação devem ser tratados de maneiras diferentes:

1. **Acesso com escopo de sessão:** resolvido automaticamente no fechamento da sessão (Seção 5.3).
2. **Concessões permanentes/recorrentes:** o Titular deve ser capaz de revogar um escopo recorrente previamente concedido a partir da interface do Agente a qualquer momento, e a revogação **DEVE** se propagar antes que a próxima requisição sob aquele escopo seja atendida, e não apenas "eventualmente". As implementações **DEVERIAN** tratar concessões permanentes como de curta duração e passíveis de reconfirmação (por exemplo, solicitar nova confirmação após $N$ dias ou $M$ requisições) em vez de indefinidas.

---

## 10. Mitigações de Deputado Confuso (Confused-Deputy)

Este é o problema em aberto mais consequente no design. O Agente de um Titular normalmente possui acesso a ferramentas e fontes de dados que vão além do que um único App está autorizado a visualizar (e-mail, calendário, outros apps conectados via MCP). Um App malicioso ou comprometido pode induzir o Agente a também extrair dados dessas *outras* fontes ao compor sua resposta — mascarando uma requisição de dados fora do escopo por meio de uma que aparenta estar dentro do escopo.

O A2AT exige que:

* A imposição de escopo do Agente **DEVE** ser avaliada em relação aos **identificadores de escopo declarados** na requisição, e não ao conteúdo do comando em texto livre do App, e **DEVE** rejeitar ou remover qualquer parte de uma resposta que exija acesso fora do escopo declarado — mesmo que o próprio modelo estivesse disposto a produzi-la.
* A imposição de escopo **DEVERIA** ocorrer como um filtro rígido na camada de acesso a ferramentas do Agente (ou seja, o modelo simplesmente não tem as outras ferramentas disponíveis durante uma iteração escopada pelo A2AT), e não como uma instrução sutil para o modelo "por favor, permaneça no escopo". Instruções ao nível do comando não constituem uma barreira de segurança.

*Esta seção deve ser sinalizada explicitamente como uma área que necessita de maior análise formal antes que qualquer implementação seja considerada pronta para produção.*

---

## 11. Agregação e Limitação de Taxa (Rate-Limiting)

Requisições autorizadas individualmente ainda podem ser combinadas ao longo do tempo para reconstruir um perfil mais amplo do que qualquer concessão única implica. O A2AT recomenda que os Agentes monitorem a **divulgação cumulativa por escopo, por App, ao longo de uma janela de tempo deslizante**, e acionem novamente o consentimento assim que um limite for ultrapassado — em vez de apenas limitar a taxa pelo volume bruto de requisições.

---

## 12. Relação com Padrões Existentes

O A2AT foi deliberadamente desenhado para não reinventar a roda. Ele compõe:

* **OAuth 2.1** para a estrutura de autorização.
* **DPoP (RFC 9449)** para tokens restritos ao remetente e resistentes a repetição.
* **Escopo de ferramentas no estilo MCP / Apps SDK** para a forma como o Agente expõe e restringe o acesso a ferramentas durante uma sessão.
* APIs padrão de **atestação de plataforma móvel** para a identidade do App.

A contribuição do A2AT não reside em uma nova criptografia, mas sim na definição da semântica de *consentimento, auditoria e deputado confuso* específica para a direção App→Agente, que nenhum dos padrões acima aborda isoladamente.

---

## 13. Caminho de Implantação e Adoção pelos Provedores

O A2AT não pode ser implantado unilateralmente por uma aplicação contra um cliente de chat não modificado — isso reproduziria o exato padrão que os provedores já se mobilizaram para encerrar (Seção 1.2). Um caminho realista de adoção exige:

1. Uma **implementação de referência e uma documentação formal** (este documento, iterado em direção a um rascunho completo) distribuídas para obter feedback de desenvolvedores já ativos no ecossistema MCP/Apps SDK.
2. Engajamento com a equipe de relações com desenvolvedores ou de padronização de ao menos um provedor, posicionando o A2AT como uma capacidade complementar ao MCP/Apps SDK (direção inversa) e não como um mecanismo concorrente ou de evasão.
3. Um projeto-piloto restrito e estritamente opcional — por exemplo, um tipo de escopo sancionado pelo provedor dentro de uma superfície existente de Apps SDK / extensão — antes de propor qualquer padronização mais ampla.

---

## 14. Considerações de Segurança (Resumo)

* A segurança de encaminhamento (*forward secrecy*) é obrigatória; o comprometimento de uma sessão não deve expor retroativamente o tráfego passado.
* Os tokens devem ser restritos ao remetente (DPoP ou equivalente), e não apenas do tipo portador (*bearer*).
* O consentimento deve ser rotulado por escopo através de um vocabulário fechado e independente do App; o texto de justificativa fornecido pelo App é apenas informativo, nunca autoritativo.
* A imposição de escopo deve ser uma barreira rígida de acesso a ferramentas, e não uma instrução contida no comando (*prompt*).
* Os logs de auditoria devem ser invioláveis e co-assinados por ambas as partes.
* A identidade do App deve ser atestada via mecanismos de plataforma antes que o consentimento seja solicitado.
* A divulgação cumulativa deve ser monitorada por escopo para prevenir ataques de agregação.
* O A2AT não protege contra um endpoint Agente comprometido, um dispositivo comprometido ou o tratamento de dados do próprio Provedor — estes permanecem fora de escopo e devem ser declarados como tal na documentação de qualquer implementação.

---

## 15. Questões em Aberto e Trabalhos Futuros

* Verificação formal da mitigação de deputado confuso (Seção 10) utilizando um framework de análise de protocolo existente (por exemplo, Tamarin ou ProVerif).
* Vocabulário de escopo padronizado — quem o governa, como é estendido e como os provedores garantem a consistência entre diferentes Agentes.
* Semântica de recuperação quando o Agente estiver offline ou tiver a taxa limitada (*rate-limited*) por seu próprio Provedor no meio de uma sessão.
* Se as sessões A2AT deveriam ser observáveis/auditáveis pelo próprio Provedor para detecção de abusos, e o que isso implica para a postura de "não ser um protocolo de confidencialidade contra o Provedor" descrita na Seção 3.

---

## Referências
1. IETF RFC 9449 — *OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)*
2. IETF RFC 6749 / OAuth 2.1 (draft) — *The OAuth Authorization Framework*
3. Anthropic — Atualização dos Termos de Serviço sobre tokens OAuth de assinatura em ferramentas de terceiros, Fevereiro–Abril de 2026.
4. OpenAI — *Apps SDK: Guide to Authentication and User Consent*
5. Especificação do Model Context Protocol (MCP).
6. Signal Protocol / Noise Protocol Framework — justificativa de design para troca de chaves efêmeras com segurança de encaminhamento.

---

*Este é um rascunho experimental (v0.1) destinado a solicitar comentários e revisões preliminares.*
