# Contexto: Automação de Triagem de Erros em Produção

Na nossa arquitetura baseada em microsserviços, utilizamos o **Sentry** para monitoramento de erros e captura de exceções em tempo real. Atualmente, os desenvolvedores precisam monitorar ativamente o Sentry, avaliar se o erro é de produção, abrir manualmente um ticket de bug no **Jira** com o stack trace (rastro do erro) e, por fim, avisar o time de plantão no **Slack**.

Esse processo manual gera atrasos na resposta a incidentes, falhas de comunicação e quebra o fluxo de trabalho (context switching). 

Para resolver isso aplicando princípios de *Clean Operations* e automação de CI/CD pipelines, vamos criar uma automação no N8N que atua como uma camada de integração (semelhante ao padrão *Ports and Adapters* da Arquitetura Hexagonal, onde o N8N orquestra a comunicação entre serviços externos sem acoplar nossa aplicação diretamente a eles).

---

## O que eu quero

Quero criar uma automação no N8N para automatizar a triagem e notificação de erros críticos ocorridos em produção.

**Público ou responsável:**
Time de Engenharia de Software (Desenvolvedores e Engenheiros de Confiabilidade - SREs que estão de plantão).

**Resultado esperado:**
Capturar automaticamente a falha, criar uma tarefa estruturada com os detalhes técnicos do erro para análise e disparar um alerta imediato no canal de comunicação da equipe, reduzindo o tempo de resposta a incidentes.

**Ferramentas envolvidas:**
Sentry (via Webhook), Jira Software e Slack.

**Fluxo desejado:**
1. Receber um payload via Webhook do Sentry contendo os detalhes de uma nova exceção (erro).
2. Filtrar e extrair as informações essenciais do JSON recebido (Nome da aplicação afetada, Mensagem do erro e URL do erro no Sentry).
3. Criar uma nova *Issue* do tipo "Bug" no Jira (no projeto "Engineering"), inserindo os detalhes técnicos e o link do Sentry na descrição.
4. Enviar uma mensagem formatada no canal `#alertas-producao` do Slack marcando o grupo `@on-call`, contendo um resumo do erro e o link direto para o ticket do Jira recém-criado.

**Regras importantes:**
- O fluxo só deve continuar se a propriedade `environment` no payload do webhook for estritamente igual a `production`. Erros de `staging` ou `development` devem ser ignorados.
- Tratar possíveis falhas de comunicação com a API do Jira: se o Jira estiver fora do ar ou retornar erro, enviar um alerta de "Fallback" direto para o Slack informando que um erro ocorreu, mas o ticket não pôde ser criado.