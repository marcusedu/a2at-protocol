# A2AT: App-to-Agent Tunnel

**Conexão segura, temporária e controlada pelo usuário entre aplicativos e agentes de IA pessoais.**

Uma proposta de protocolo que permite que qualquer aplicação peça ajuda ao agente de IA do próprio usuário (ChatGPT, Claude, Gemini etc.) **sem precisar de chaves de API**, sem forçar o usuário a gerenciar chaves e com fortes garantias de privacidade e segurança.

---

## A Motivação

Eu precisava integrar IA em uma aplicação. Em determinado momento, minha chave vazou e eu tive prejuízo financeiro. 

Além disso, percebi que forçar o usuário a pagar por uma IA dentro da minha aplicação (quando ele já pode ter GPT Plus, Claude Pro ou Gemini Advanced) não fazia sentido — tanto para ele quanto para mim.

A solução atual (gerar prompt + usuário copiar e colar) funciona, mas cria muita fricção.

## O que o A2AT propõe

Criar um **túnel temporário** entre o aplicativo e o agente de IA do usuário, com:

- Sessões efêmeras e seguras (forward secrecy)
- Consentimento granular mostrado pelo próprio Agente
- Auditoria co-assinada e à prova de adulteração
- Proteção contra apps maliciosos e vazamento de dados
- Sem nunca expor chaves de API

## Benefícios

**Para desenvolvedores** (principalmente indies e pequenos times):
- Reduz drasticamente riscos de segurança e financeiros
- Facilita integração com IA
- Melhora a experiência do usuário → mais retenção

**Para usuários**:
- Usa a IA que ele já paga e confia
- Controle total sobre o que é compartilhado
- Experiência muito mais fluida

## Status

**Draft v0.1** — Este é um protocolo aberto em fase inicial, buscando feedbacks da comunidade, provedores de IA e desenvolvedores.

[Leia o Whitepaper completo](./a2at-whitepaper-pt-br.md)

## Como Participar

- ⭐ Dê uma estrela se você acha a ideia valiosa
- Compartilhe casos de uso
- Contribua com o design do protocolo
- Ajude a espalhar para builders de produtos de IA

---

**Nascido de uma dor real.**  
Vamos construir um futuro melhor de integração entre aplicativos e agentes de IA pessoais.
