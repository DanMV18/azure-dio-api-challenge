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
Com certeza. Para elevar este `README.md` a um nível de documentação de arquitetura corporativa (padrão *Enterprise*), vamos adicionar seções críticas sobre **Governança de API**, **Ciclo de Vida (SDLC)**, **Resiliência** e **Detalhamento do Proxy Reverso**.

Aqui está a versão expandida, mais técnica e rigorosa:

---

## 🏛️ Detalhamento da Arquitetura de Rede

A solução utiliza uma topologia de **Hub-and-Spoke** para garantir que nenhum serviço de backend seja exposto diretamente à rede pública (Internet).

### 1. O Papel do Gateway como Proxy Reverso
O **Azure API Management (APIM)** atua como a única porta de entrada. Ele executa as seguintes funções de Proxy Reverso:
* **Terminação TLS:** O tráfego criptografado é decriptado no gateway para inspeção de políticas e re-criptografado para o transporte interno.
* **Encaminhamento de Backend (URL Rewriting):** Mapeia URLs externas amigáveis (ex: `/v1/orders`) para microserviços específicos no App Service (ex: `internal-order-svc-001.azurewebsites.net`).
* **Ocultação de Infraestrutura:** Remove cabeçalhos de resposta que revelam a tecnologia de backend (como `X-AspNet-Version` ou `Server`), dificultando o reconhecimento por agentes maliciosos.

### 2. Camada de Computação: App Services & Planos de Isolamento
* **App Service Environment (ASE):** (Opcional) Para isolamento total em nível de rede virtual (VNET).
* **VNET Integration:** Permite que o App Service acesse recursos internos (Bancos de Dados, Key Vaults) sem sair da rede privada da Microsoft.

---

## 🔐 Estratégia de Segurança "Zero Trust"

A segurança não é baseada apenas no perímetro, mas em cada transação:

### Autenticação em Camadas
1.  **Nível de Gateway (APIM):** Validação de `OIDC (OpenID Connect)` contra o **Microsoft Entra ID**. O gateway rejeita qualquer chamada sem um `Bearer Token` válido antes mesmo de tocar o código do desenvolvedor.
2.  **Chaves de Produto (Subscriptions):** Implementação de cotas por parceiro, permitindo revogação granular de acesso sem afetar outros consumidores.
3.  **Identidade Gerenciada (MSI):** O código no App Service não possui "Connection Strings" com senhas. Ele utiliza o token de identidade do recurso para se autenticar no SQL Server e no Key Vault.

### Criptografia e Segredos
* **Azure Key Vault:** Armazenamento centralizado de certificados SSL/TLS e chaves de criptografia.
* **Políticas de Acesso:** Apenas o App Service em runtime tem permissão de "Get" nos segredos necessários.

---

## 🛠️ Área do Desenvolvedor & Governança (DevPortal)

O sucesso de uma estratégia de API depende da **Developer Experience (DX)**.
* **Self-Service:** Desenvolvedores podem se cadastrar e solicitar acesso a "Produtos" (agrupamentos de APIs).
* **Documentação Automática:** O portal renderiza o Swagger/OpenAPI em tempo real, fornecendo exemplos de código em múltiplas linguagens (Curl, C#, Java, Python).
* **Mocking de Respostas:** O APIM é configurado para retornar respostas estáticas (Mocks) enquanto o backend ainda está em desenvolvimento, permitindo que o time de Frontend trabalhe em paralelo.

---

## 🔄 Ciclo de Vida e CI/CD (DevOps)

O processo de deploy é totalmente automatizado via **GitHub Actions** ou **Azure Pipelines**:

| Estágio | Ferramenta | Descrição |
| :--- | :--- | :--- |
| **Infra** | Terraform / Bicep | Provisionamento do APIM e Web Apps como código. |
| **Build** | .NET / Node.js CLI | Compilação e execução de Testes Unitários. |
| **Deploy** | ZipDeploy / WebDeploy | Publicação nos *Deployment Slots* (Staging). |
| **Gate** | API Tests | Testes de fumaça (Smoke Tests) antes do Swap para Produção. |
| **APIM Sync** | Azure CLI | Importação automática do novo arquivo `swagger.json` para o Gateway. |

---

## 📈 Resiliência e Monitoramento Avançado

### Health Checks
O Gateway monitora a saúde dos App Services através de probes HTTP. Se uma instância do backend falha, o APIM para de enviar tráfego para ela automaticamente (Circuit Breaker pattern).

### Application Insights (Kusto Queries)
Utilizamos telemetria correlacionada para rastrear gargalos:
```kusto
// Query para identificar latência média por operação de API
requests
| where timestamp > ago(24h)
| summarize avg(duration) by name, resultCode
| order by avg_duration desc
```

---

## 📝 Como contribuir
1.  Faça o Fork do repositório.
2.  Crie uma branch para sua Feature (`git checkout -b feature/NovaPoliticaSeguranca`).
3.  Abra um Pull Request detalhando as mudanças na infraestrutura ou nas políticas de Gateway.

---
