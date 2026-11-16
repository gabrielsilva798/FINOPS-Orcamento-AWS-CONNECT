# FINOPS | Service Desk de TI na AWS: Amazon Connect, Lex, Cases e Lambda

Fluxo de atendimento do **Service Desk de TI** (voz e chat) em Amazon Connect, com o **modelo de custos e as práticas de FinOps** para um cenário de **1.000 ligações por dia**.

> **Status:** fluxo de testes (`Fluxo-ServiceDesk-TI-TESTE-Protocolo`), publicado, `us-east-1`.
> **Escopo de custo:** service desk **interno**; telefonia (DID e minutos de entrada) **não incluída**.

---

## 1. Visão geral

O funcionário entra em contato (voz ou chat), um bot **Amazon Lex** identifica a necessidade e o fluxo executa a parte repetitiva do atendimento.

| Intent do Lex | O que o fluxo faz | Destino |
|---|---|---|
| `PasswordReset` | Localiza o funcionário, confirma identidade por **OTP por e-mail**, abre case de prioridade alta | Fila de **Segurança** |
| `NetworkIssue` | Localiza o funcionário, abre case de prioridade média | Fila de **Infraestrutura e Redes** |
| `HardwareRequest` | Localiza o funcionário, abre case de prioridade média | Fila de **Hardware** |
| `ConsultarChamado` | Consulta o case pelo protocolo e informa o status | Autoatendimento |
| `OutOfScope` (ou sem correspondência) | Encerra o contato | Sem atendimento |

**Componentes:** Amazon Connect (fluxo, voz, filas), Lex V2 (intents e slots), Customer Profiles (cadastro por matrícula), Connect Cases (chamados com protocolo), Lambda `edn-validar-identidade-otp` (envia/valida OTP) e `edn-consultar-protocolo` (recupera o protocolo), Contact Lens (análise e sentimento), S3 (gravações e análises), CloudWatch Logs (logs do fluxo, Lex e Lambdas) e SES/DynamoDB (apoio ao OTP).

---

## 2. O fluxo de atendimento

```mermaid
flowchart TD
    A(["Contato entra (voz ou chat)"]) --> B["Ativa logs do fluxo, gravação e Contact Lens"]
    B --> C["Define voz Ana Paula e idioma pt-BR"]
    C --> D{{"Triagem com Amazon Lex"}}

    D -->|"PasswordReset"| S1["Busca perfil pela matrícula"]
    D -->|"NetworkIssue"| R1["Busca perfil pela matrícula"]
    D -->|"HardwareRequest"| H1["Busca perfil pela matrícula"]
    D -->|"ConsultarChamado"| K1["Busca perfil pela matrícula"]
    D -->|"OutOfScope ou sem correspondência"| X(["Encerra o contato"])

    S1 -->|"encontrou"| S2["Lambda envia OTP por e-mail"]
    S1 -->|"não encontrou ou erro"| QS
    S2 --> S3{"Canal"}
    S3 -->|"Voz"| S4["Cliente digita 6 dígitos (DTMF)"]
    S3 -->|"Chat"| S5["Lex coleta o código"]
    S4 --> S6["Lambda verifica OTP"]
    S5 --> S6
    S6 --> S7{"Código válido?"}
    S7 -->|"sim"| S8["Cria case (prioridade Alta)"]
    S7 -->|"não"| S9["Aviso: identidade não confirmada"] --> X
    S8 --> S10["Lambda consulta o protocolo"]
    S10 --> S11["Informa o protocolo"]
    S11 --> QS
    S8 -->|"falha"| S12["Aviso de falha no registro"] --> QS

    R1 -->|"encontrou"| R2["Cria case (prioridade Média)"]
    R1 -->|"não encontrou, duplicado ou erro"| QR
    R2 --> R3["Lambda consulta o protocolo"]
    R3 --> R4["Informa o protocolo"]
    R4 --> QR

    H1 -->|"encontrou"| H2["Cria case (prioridade Média)"]
    H1 -->|"não encontrou, duplicado ou erro"| QH
    H2 --> H3["Lambda consulta o protocolo"]
    H3 --> H4["Informa o protocolo"]
    H4 --> QH

    K1 -->|"encontrou"| K2["Busca o case por protocolo e perfil"]
    K1 -->|"não encontrou, duplicado ou erro"| K4["Mensagem de erro"] --> X
    K2 --> K3{"Status"}
    K3 -->|"open"| K5["Informa: Aberto"] --> X
    K3 -->|"closed"| K6["Informa: Encerrado"] --> X
    K3 -->|"outro"| K7["Informa o status recebido"] --> X
    K2 -->|"não encontrou ou erro"| K4

    QS["Define fila Segurança"] --> T
    QR["Define fila Infraestrutura e Redes"] --> T
    QH["Define fila Hardware"] --> T
    T["Transfere para a fila"] -->|"fila sem capacidade"| Z["Aviso de indisponibilidade"] --> X
```

**Pontos principais**

- **Todo contato:** logs do fluxo, gravação de voz e Contact Lens ativos; voz `ANA_PAULA` em `pt-BR`. A **redação de dados sensíveis está desligada**.
- **Senha:** o Lex captura a matrícula, a Lambda envia um OTP de 6 dígitos (validade de 5 min) e o valida. Se válido, cria case Alta, informa o protocolo e transfere para Segurança. Se inválido, informa e encerra (uma tentativa por ligação). Se o perfil não for achado ou o case falhar, vai para a fila sem protocolo.
- **Rede e hardware:** busca o perfil, cria case Média, informa o protocolo e transfere. Sem perfil (ou duplicado), vai para a fila **sem case**.
- **Consulta:** valida perfil, busca o case por `customer_id` + `reference_number` e informa assunto e status. É autoatendimento puro.
- **Erros:** falha técnica ou fila sem capacidade geram aviso e encerramento; Lambda tem timeout de 8 s e o fluxo segue para a fila ou encerra, conforme o ramo.

---

## 3. Modelo de custos (1.000 ligações/dia)

### 3.1 Premissas

| Premissa | Valor |
|---|---|
| Ligações | 30.000/mês |
| Mix de intents (estimativa) | Senha 30%, Rede 25%, Hardware 15%, Consulta 20%, Fora de escopo 10% |
| Duração faturável (estimativa) | Senha 9 min, Rede 11, Hardware 8, Consulta 2, Fora de escopo 1 → **média 7,15 min** → 214.500 min/mês |
| Perfil encontrado / OTP validado | 95% / 85% |
| Cases criados | ~622/dia → ~18.700/mês |
| Lex | 5 requisições de fala por ligação → 150.000/mês |
| Gravação | ~1 MB/min → ~210 GB/mês |

### 3.2 Custo mensal (cenário base)

| Item | Cálculo | US$/mês | % |
|---|---|---:|---:|
| Amazon Connect, voz (plano Customer, IA inclusa) | 214.500 min × 0,038 | 8.151 | 73,1% |
| Cases | 18.700 × 0,12 | 2.240 | 20,1% |
| Lex V2 (fala) | 150.000 × 0,004 | 600 | 5,4% |
| Customer Profiles | 900/dia × 30 × 0,005 | 135 | 1,2% |
| Lambda | ~36 mil invocações × 0,5 s × 512 MB | ~0,20 | < 0,1% |
| S3 (gravações + Contact Lens) | ~210 GB/mês | ~5 no mês 1; ~58 no mês 12 e ~173 no mês 36, sem lifecycle | 0,0% |
| CloudWatch Logs (fluxo, Lex, Lambda) | ~5 GB | ~5 | < 0,1% |
| Extras (SES para o OTP, DynamoDB) | estimativa | ~10 | 0,1% |
| **Total (mês 1)** | | **≈ 11.150** | 100% |
| **Custo por ligação** | 11.150 ÷ 30.000 | **≈ US$ 0,37** | |

**Pareto:** voz + Cases + Lex somam **~99%** do custo. O S3 é irrelevante hoje, mas **cresce todo mês** sem retenção.

### 3.3 Custo unitário por intent (custo direto por ligação)

| Intent | Voz | Lex | Perfil | Case | **Por ligação** | **Total/mês** |
|---|---:|---:|---:|---:|---:|---:|
| Senha | 0,342 | 0,020 | 0,005 | 0,097 | **0,464** | 4.175 |
| Rede | 0,418 | 0,020 | 0,005 | 0,114 | **0,557** | 4.178 |
| Hardware | 0,304 | 0,020 | 0,005 | 0,114 | **0,443** | 1.993 |
| Consulta | 0,076 | 0,020 | 0,005 | n/a | **0,101** | 606 |
| Fora de escopo | 0,038 | 0,020 | n/a | n/a | **0,058** | 174 |
| Soma dos custos diretos | | | | | | 11.126 |
| Custos compartilhados (S3, CloudWatch, Lambda, extras), a ratear | | | | | | ~20 |

Valores de coluna em US$.

### 3.4 Sensibilidade à duração média

| Cenário | Duração média | Minutos/mês | Total/mês (US$) | Por ligação (US$) |
|---|---|---:|---:|---:|
| Baixo | 5 min | 150.000 | ≈ 8.700 | 0,29 |
| **Base** | **7,15 min** | **214.500** | **≈ 11.150** | **0,37** |
| Alto | 10 min | 300.000 | ≈ 14.400 | 0,48 |

**Regra prática:** cada **1 minuto** a menos na duração média economiza cerca de **US$ 1.140/mês**.

### 3.5 Não incluído

- Telefonia (DID e minutos de entrada). Se as ligações forem pelo navegador/app (**in-app calling**), pode haver cobrança de US$ 0,010/min (~US$ 2.145/mês).
- Impostos, conversão para reais, suporte AWS, CloudTrail e KMS.
- Agentes, implantação, desenvolvimento e o canal de chat (o cenário considera só voz).

---

## 4. Práticas FinOps (Informar → Otimizar → Operar)

| Fase | Práticas |
|---|---|
| **Informar** | Padrão de tags (`Project=ServiceDeskTI`, `Environment`, `Owner`, `CostCenter`, `Service`); ambiente de teste separado de produção; showback por fila a partir dos CTRs; acompanhar custo por ligação, por case e por intent |
| **Otimizar** | Focar em voz, Cases e Lex (99% do custo); automatizar o reset de senha; comparar planos Customer x Basic + adicionais; lifecycle no S3 e retenção de 30 a 90 dias no CloudWatch |
| **Operar** | AWS Budgets de ~US$ 11.500/mês (alertas em 80% previsto e 100% real); referência diária de ~US$ 372 (alerta acima de ~US$ 450); Cost Anomaly Detection por tag; revisão mensal com TI e Finanças |

**KPIs de referência:** custo por ligação US$ 0,37; duração média 7,15 min; ~622 cases/dia; 5 turnos de Lex por ligação; variação real x previsto dentro de ±10%.

---

## 5. Oportunidades de otimização

| # | Ação | Impacto estimado | Esforço |
|---|---|---|---|
| 1 | **Automatizar o reset de senha** pela Lambda (reduzir de ~9 para ~3,5 min) | ~US$ 1.900/mês | Médio |
| 2 | **Reduzir a duração média** (base de conhecimento, melhor roteamento) | **US$ 1.140/mês por minuto** | Médio |
| 3 | **Evitar cases duplicados** | **US$ 224/mês a cada 10% evitados** | Baixo |
| 4 | **Menos turnos do Lex** | **US$ 120/mês por requisição a menos por ligação** | Baixo |
| 5 | **Avaliar o plano Connect Basic + adicionais** | ~US$ 500/mês (terceiros; validar) | Baixo |
| 6 | **Lifecycle no S3** e retenção de logs | Evita crescimento linear (US$ 58 no mês 12, US$ 173 no mês 36) | Baixo |

---

## 6. Segurança, LGPD e operação

- **Dados sensíveis:** gravação ativa e redação do Contact Lens desligada; matrícula e OTP podem ficar em gravações e transcrições. Ativar a redação, tratar o OTP só por DTMF/mascarado e definir retenção conforme a LGPD.
- **Tentativas de OTP:** limitar por matrícula na Lambda e expirar o código.
- **Rede/hardware sem perfil:** o contato segue sem case e sem protocolo; considerar criar o case mesmo assim.
- **Chamadas simultâneas:** verificar a cota da instância antes de produção.
- **Identificadores:** não publicar ID da conta, instância, ARNs de filas e do bot em repositório público; usar placeholders (`<ACCOUNT_ID>`, `<INSTANCE_ID>`, `<QUEUE_ID>`, `<BOT_ALIAS_ID>`).

---

## 7. Limitações e validações

Mix, durações, taxas de perfil e OTP, turnos do Lex e MB/min são **estimativas**; substituir por dados reais (CTR, relatórios do Connect e CloudWatch). Lex tratado como custo separado e Customer Profiles com cobrança assumida (US$ 135/mês); preços de Lambda, S3 e CloudWatch não foram reconferidos.

**Checklist antes de aprovar o orçamento**

- [ ] Plano contratado (Customer ou Basic) e preços atuais do Amazon Connect
- [ ] Simulação no AWS Pricing Calculator com estas premissas
- [ ] Duração média e mix de intents reais
- [ ] Cobrança de Customer Profiles e de Lex na conta
- [ ] Forma de entrada das ligações (PABX/SIP, in-app) e cobrança de comunicação

**Fontes**

- Preços e documentação oficiais da AWS (Connect, Lex, Lambda, S3, CloudWatch)
- [Amazon Connect FinOps: Cost Optimization & Usage Audit](https://aws.amazon.com/marketplace/pp/prodview-y37q4y3moyaxe)
- [AWS FinOps Agent (Preview)](https://aws.amazon.com/pt/finops-agent/)
- [Getting started with AWS FinOps Agent](https://docs.aws.amazon.com/finops-agent/latest/userguide/getting-started.html)
- Estrutura do fluxo: arquivo
