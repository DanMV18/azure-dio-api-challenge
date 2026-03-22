---

# Enterprise API Gateway & Serverless Architecture on Microsoft Azure

## 📑 Visão Geral
Este repositório documenta a implementação de um ecossistema de APIs escalável, utilizando o **Azure API Management (APIM)** como o orquestrador central e o **Azure App Services** como a camada de computação. 

A arquitetura foi desenhada para suportar alta disponibilidade, segurança de confiança zero (*Zero Trust*) e uma experiência de desenvolvedor fluida através do Portal do Desenvolvedor automatizado.

---

## 🏗️ Arquitetura Detalhada

### 1. Camada de Exposição: Azure API Management (APIM)
O APIM não é apenas um pass-through; ele funciona como um **Proxy Reverso Inteligente** e uma camada de governança.
* **Abstração de Backend:** O cliente nunca conhece a URL real `https://meu-app-service.azurewebsites.net`. Ele interage apenas com `https://api.empresa.com`.
* **Versionamento:** Implementação de revisões e versões (Header ou Path-based) para garantir que mudanças no contrato não quebrem clientes legados.
* **Área do Desenvolvedor (Developer Portal):** Um portal CMS integrado onde desenvolvedores internos e externos podem se cadastrar, obter chaves de subscrição e testar endpoints via console interativo.

### 2. Camada de Processamento: App Services
Nesta arquitetura, utilizamos o **Azure App Service** (Web Apps) para hospedar as APIs.
* **Isolamento:** Os apps são implantados em um *App Service Plan* dedicado, permitindo escalonamento horizontal (Auto-scaling) baseado em métricas de CPU e Memória.
* **Deployment Slots:** Utilização de slots de *Staging* para Blue/Green deployment, garantindo que novas versões da API sejam testadas no ambiente produtivo antes do swap final.

---

## 🛠️ Configuração do Gateway e Proxy Reverso

O fluxo da requisição segue uma lógica de processamento de pipeline:

1.  **Frontend:** O APIM recebe a requisição HTTPS, valida o certificado SSL e a chave de subscrição.
2.  **Inbound Processing:** Execução de políticas (XML) para sanitização de inputs e verificação de limites.
3.  **Backend (Proxy):** O APIM altera o cabeçalho `Host` e encaminha a chamada para o App Service privado.
4.  **Outbound Processing:** Antes de retornar ao usuário, o APIM pode remover cabeçalhos sensíveis do servidor (como `X-Powered-By`) para evitar exposição de tecnologia.

---

## 🔒 Autenticação, Autorização e Segurança

A segurança é o pilar central desta implementação, estruturada em três níveis:

### A. Proteção de Borda (Edge Security)
* **Web Application Firewall (WAF):** Integrado ao Azure Front Door ou Application Gateway (opcional) para proteger contra ataques SQL Injection e Cross-Site Scripting (XSS).
* **Criptografia:** TLS 1.2 ou 1.3 obrigatório em todos os pontos de extremidade.

### B. Identidade e Acesso
* **OAuth 2.0 & Azure AD:** Integração com o Microsoft Entra ID (antigo Azure AD). O APIM valida o `access_token` (JWT) enviado no header `Authorization`.
* **Managed Identities:** O App Service utiliza uma identidade gerenciada no Azure para autenticar-se em outros serviços (como SQL Database ou Storage) sem armazenar strings de conexão ou senhas no `appsettings.json`.

### C. Isolamento de Rede
* **VNET Integration:** O App Service é injetado em uma Rede Virtual (VNET).
* **Access Restrictions:** O firewall do App Service é configurado com uma regra de prioridade máxima que permite tráfego **apenas** do endereço IP estático do Azure API Management.

---

## ⚡ Políticas (Policies) Customizadas
As políticas são o "cérebro" do Gateway. Abaixo, exemplos das lógicas implementadas neste repositório:

### Controle de Vazão (Throttling)
Impede que um único consumidor derrube a infraestrutura:
```xml
<rate-limit-by-key calls="500" renewal-period="60" counter-key="@(context.Subscription.Id)" />
```

### Cache de Resposta
Reduz a latência e o custo de computação no backend para dados que mudam raramente:
```xml
<outbound>
    <cache-store duration="3600" />
</outbound>
```

---

## 📊 Observabilidade e Governança

Para garantir o SLA do serviço, utilizamos a stack de monitoramento do Azure:

1.  **Application Insights:** Rastreamento distribuído (*Distributed Tracing*). É possível ver o caminho exato da requisição desde o APIM até o banco de dados.
2.  **Log Analytics:** Consultas KQL para auditoria. Exemplo de busca por erros:
    ```kusto
    ApiManagementGatewayLogs
    | where IsRequestSuccess == false
    | summarize count() by TotalTime, ApiId
    ```
3.  **Azure Advisor:** Recomendações automáticas de segurança e otimização de custos para os recursos de API.

---

## 🚀 Como fazer o Deploy

1.  **Infraestrutura como Código (IaC):** Utilize os templates Bicep ou Terraform localizados na pasta `/infra`.
2.  **CI/CD Pipeline:** * O GitHub Actions faz o build da aplicação.
    * Publica o artefato no App Service Slot.
    * Atualiza a definição OpenAPI (Swagger) no APIM via CLI do Azure.

---
> **Nota:** Este projeto segue as melhores práticas do *Azure Well-Architected Framework*.

http://googleusercontent.com/interactive_content_block/0
