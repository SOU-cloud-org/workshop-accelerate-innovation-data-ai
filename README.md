# Lab: Logistics Data Agent — Microsoft Fabric

Guia passo a passo do lab prático.

```
CSV  →  Lakehouse  →  Semantic Model  →  Reports + Ontology + Data Agent
```

Um único modelo semântico alimentando Analytics e IA.

## Visão geral

- **Cenário:** análise de performance logística nacional.
- **Dados:** entregas, produtos, centros de distribuição, transportadoras e calendário.
- **Objetivo:** responder perguntas sobre SLA, atrasos, devoluções, receita e custos operacionais.
- **Resultado:** dois relatórios, uma Ontology e um Fabric Data Agent conectados ao mesmo modelo semântico.

> O Data Agent não substitui a modelagem. Ele reutiliza relacionamentos, medidas, descrições e contexto já preparados no Fabric.

## Pré-requisitos

- [ ] Workspace do Fabric disponível.
- [ ] Pacote com os cinco CSVs extraído.
- [ ] Permissão para criar Lakehouse, Semantic Model, Reports, Ontology e Data Agent.
- [ ] Este guia aberto para copiar DAX, descrições, prompts e instruções.

---

## 1. Criar o Lakehouse e carregar os dados

- Criar o Lakehouse `LH_Logistics`.
- Carregar os cinco CSVs como novas tabelas.
- Confirmar que os objetos aparecem na área **Tabelas**, não apenas em **Arquivos**.

```
FactDeliveries.csv            → FactDeliveries
DimDate.csv                   → DimDate
DimProduct.csv                → DimProduct
DimDistributionCenter.csv     → DimDistributionCenter
DimCarrier.csv                → DimCarrier
```

---

## 2. Criar o modelo semântico e os relacionamentos

- Criar um novo modelo semântico a partir das cinco tabelas.
- Abrir a **Exibição do modelo**.
- Criar relacionamentos 1:* com filtro das dimensões para a fato.

```
DimDate[DateKey]                            → FactDeliveries[DateKey]
DimProduct[ProductID]                       → FactDeliveries[ProductID]
DimDistributionCenter[DistributionCenterID] → FactDeliveries[DistributionCenterID]
DimCarrier[CarrierID]                       → FactDeliveries[CarrierID]
```

**Validação:** `FactDeliveries` deve ficar no centro do esquema estrela, conectada às quatro dimensões.

---

## 3. Criar as medidas DAX

```dax
Total Revenue =
SUM(FactDeliveries[Revenue])
```

```dax
Total Deliveries =
SUM(FactDeliveries[DeliveredOrders])
```

```dax
Total Delays =
SUM(FactDeliveries[DelayedOrders])
```

```dax
Delay Rate % =
DIVIDE([Total Delays], [Total Deliveries])
```

```dax
On-Time SLA % =
1 - [Delay Rate %]
```

```dax
Total Returns =
SUM(FactDeliveries[ReturnedOrders])
```

```dax
Return Rate % =
DIVIDE([Total Returns], [Total Deliveries])
```

```dax
Total Penalty Cost =
SUM(FactDeliveries[PenaltyCost])
```

```dax
Average Delivery Days =
AVERAGE(FactDeliveries[AverageDeliveryDays])
```

```dax
Revenue per Delivery =
DIVIDE([Total Revenue], [Total Deliveries])
```

**Formatação:**

- Percentual: `Delay Rate %`, `On-Time SLA %`, `Return Rate %`.
- Moeda: `Total Revenue`, `Total Penalty Cost`, `Revenue per Delivery`.
- O SLA geral esperado é aproximadamente **89,86%**.

---

## 4. Adicionar descrições

### Tabelas

**FactDeliveries**

```
Contains logistics performance metrics for completed deliveries across the company distribution network. Each record aggregates delivery volume, revenue, delays, returns and operational costs associated with a specific product, carrier, distribution center and date.
```

**DimProduct**

```
Represents products delivered through the logistics network. Products are grouped into categories and brands for operational and financial analysis.
```

**DimDistributionCenter**

```
Represents company distribution centers responsible for fulfilling customer deliveries. Used to analyze logistics performance by location and region.
```

**DimCarrier**

```
Represents logistics providers responsible for transporting products to customers. Used for SLA and operational performance analysis.
```

**DimDate**

```
Calendar dimension used for time-based analysis including trends, monthly performance and year-over-year comparisons.
```

### Medidas principais

| Medida | Descrição |
| --- | --- |
| Total Revenue | Total revenue generated from delivered orders. |
| On-Time SLA % | Percentage of deliveries completed within the expected delivery target. |
| Delay Rate % | Percentage of deliveries that experienced delays. |
| Return Rate % | Percentage of delivered orders that were returned. |
| Total Penalty Cost | Total operational penalties associated with delayed deliveries. |

---

## 5. Configurar o Prepare Data for AI

### Simplificar a visão para IA

- Manter métricas, atributos descritivos e campos usados em análise.
- Ocultar ou despriorizar `DeliveryID`, `DateKey`, `ProductID`, `CarrierID` e `DistributionCenterID`.
- Manter `Product`, `Category`, `Brand`, `DistributionCenter`, `City`, `State`, `Region`, `Carrier`, `ServiceType`, `Date`, `Year`, `Quarter`, `Month` e `YearMonth`.

### Instruções do modelo

```
This semantic model represents a national logistics operation.
Distribution Centers may also be called DCs or Warehouses.
Carriers may also be referred to as Logistics Providers.
When evaluating operational performance prioritize:
1. On-Time SLA %
2. Delay Rate %
3. Return Rate %
4. Total Penalty Cost
Always use existing business measures whenever possible.
Provide concise business-oriented explanations.
```

**Sinônimos:** quando não houver um campo específico, registre os termos alternativos nas descrições ou instruções:

- Distribution Center = DC, Warehouse, Distribution Hub
- Carrier = Logistics Provider, Delivery Partner, Logistics Operator

---

## 6. Gerar e revisar a Ontology

- Usar **Gerar Ontology** a partir do modelo semântico.
- Validar os entity types para `factdeliveries`, `dimcarrier`, `dimdate`, `dimdistributioncenter` e `dimproduct`.
- Confirmar propriedades e relacionamentos inferidos.

A geração automática cria a estrutura inicial. Nos próximos passos, validamos identidade, bindings, metadados e relacionamentos para que a Ontology represente corretamente o cenário logístico.

### 6.1 Definir a chave da factdeliveries

*Por quê:* definimos uma chave única para que cada entrega possa ser identificada como uma instância da entidade.

- Abrir a entidade `factdeliveries`.
- Em **Entity type key**, selecionar `DeliveryID`.
- Mapear a propriedade `DeliveryID` para a coluna `DeliveryID` da tabela `factdeliveries`.
- Salvar a configuração.

> Se aparecer “Entity type key missing”, a configuração de `DeliveryID` é obrigatória para salvar corretamente os bindings.

### 6.2 Revisar propriedades Bound e Unbound

*Por quê:* verificamos quais propriedades estão ligadas diretamente a colunas da fonte e quais existem apenas como propriedades locais da Ontology.

```
Bound / factdeliveries: propriedade vinculada a uma coluna real da tabela, como DeliveryID, CarrierID, Revenue e PenaltyCost.
Unbound: propriedade sem binding direto com uma coluna da fonte nessa configuração, como Delay_Rate_, On-Time_SLA_ e Total_Revenue. Unbound não significa automaticamente que a propriedade está incorreta; indica apenas ausência de vínculo direto.
```

### 6.3 Adicionar metadados às entidades

*Por quê:* adicionamos descrições e sinônimos para aproximar os nomes técnicos do vocabulário usado nas perguntas de negócio.

```
dimcarrier
Description: Transportation provider responsible for delivering products across the logistics network.
Synonyms: Carrier; Transporter; Logistics Provider; Transport Company; Shipping Company

dimdistributioncenter
Description: Facility responsible for storing inventory and dispatching products across the logistics network.
Synonyms: Distribution Center; Warehouse; DC; Distribution Hub; Storage Facility

dimproduct
Description: Product being transported and delivered through the logistics network.
Synonyms: Product; Item; SKU; Goods; Merchandise; Material
```

### 6.4 Adicionar descrições às propriedades principais

*Por quê:* descrevemos as métricas para que o agente consiga associar perguntas em linguagem natural aos indicadores corretos. Nesta experiência, as propriedades aceitam descrição, mas não sinônimos.

```
AverageDeliveryDays: Average number of days required to complete a delivery.
Delay_Rate_: Percentage of deliveries completed after the expected delivery date.
Revenue: Revenue generated by an individual delivery.
Total_Revenue: Total revenue generated from deliveries.
On-Time_SLA_: Percentage of deliveries completed within the agreed service level target.
PenaltyCost: Financial penalties incurred due to delayed or non-compliant deliveries.
```

### 6.5 Validar os relacionamentos

*Por quê:* confirmamos que a entidade de entregas consegue navegar até data, produto, centro de distribuição e transportadora durante as análises.

```
factdeliveries_has_dimdate
factdeliveries_has_dimproduct
factdeliveries_has_dimdistributioncenter
factdeliveries_has_dimcarrier
```

> O Graph Explorer pode continuar exibindo apenas as dimensões como nós e **Edges (0)**. Neste lab, não ajuste manualmente os nós. A validação funcional será feita no Data Agent usando a Ontology como fonte.

---

## 7. Criar o Operations Dashboard com Copilot

Prompt:

```
Create an operational logistics dashboard for business users.
Include the following KPI cards:
- Total Revenue
- On-Time SLA %
- Delay Rate %
- Total Penalty Cost
Add visualizations that help identify logistics performance issues:
1. Delay Rate % by Carrier
2. On-Time SLA % by Distribution Center
3. Total Penalty Cost by Carrier
4. Monthly trend of On-Time SLA %
5. Revenue by Region
Organize the report in a clean executive layout.
Use the measures already available in the semantic model whenever possible.
Prioritize business-friendly visualizations and avoid technical views.
```

- Revisar se o Copilot utilizou as medidas oficiais.
- Salvar como **Operations Dashboard**.

---

## 8. Criar o Executive Dashboard com Copilot

Prompt:

```
Create an executive logistics performance dashboard focused on business outcomes and financial impact.
Include the following KPI cards:
- Total Revenue
- Total Returns
- Return Rate %
- Revenue per Delivery
Add visualizations that provide an executive view of the business:
1. Total Revenue by Product Category
2. Total Revenue by Region
3. Return Rate % by Product Category
4. Monthly Revenue Trend
5. Top Product Categories by Revenue
Organize the report in a clean executive layout suitable for senior leadership.
Use the measures already available in the semantic model whenever possible.
Prioritize business-friendly visualizations and avoid technical operational details.
```

- Salvar como **Executive Dashboard**.
- Confirmar que os dois relatórios aparecem no workspace.

---

## 9. Criar o Fabric Data Agent

**Nome**

```
Logistics Operations Analyst
```

**Descrição**

```
AI assistant specialized in logistics operations, delivery performance, carrier analysis and operational KPIs.
```

**Instruções**

```
You are a logistics operations analyst.
Your objective is to help business users understand logistics performance across carriers, distribution centers, products and regions.
Always prioritize existing business measures.
When analyzing performance, prioritize:
1. On-Time SLA %
2. Delay Rate %
3. Return Rate %
4. Total Penalty Cost
5. Total Revenue
Distribution Centers may also be referred to as Warehouses or DCs.
Carriers may also be referred to as Logistics Providers or Delivery Partners.
Provide concise business-oriented explanations.
When identifying problems, explain possible operational impacts and recommendations.
```

- Adicionar o modelo semântico como fonte.
- Adicionar também a Ontology criada no passo 6 como fonte de dados do agente. Selecionar a **Ontology**, não o item separado de Graph Model/Graph Explorer.
- Salvar e abrir o chat de testes.

---

## 10. Testar o Data Agent

```
Which carrier has the highest revenue?
Which distribution center has the highest delay rate?
What is the average delivery time by carrier?
Which products generate the most revenue?
Compare the top 3 carriers by revenue and on-time delivery performance.
```

**Resultados de referência:** o On-Time SLA geral deve ficar próximo de **89,86%**. NorteLog tende a apresentar maior taxa de atraso e CD Recife tende a aparecer entre os piores resultados operacionais.

---

## 11. Abrir a visão de linhagem

- Abrir a **Exibição de linhagem** do workspace.
- Verificar Lakehouse, modelo semântico, dois relatórios, Ontology e Data Agent.
- Revisar as dependências e a reutilização do mesmo conhecimento entre os ativos.

---

## Checklist final

- [ ] Cinco tabelas carregadas no Lakehouse.
- [ ] Quatro relacionamentos ativos.
- [ ] Dez medidas criadas e formatadas.
- [ ] Descrições e Prepare Data for AI configurados.
- [ ] Ontology gerada, enriquecida e adicionada como fonte do Data Agent.
- [ ] Dois relatórios salvos.
- [ ] Data Agent conectado e testado.
- [ ] Visão de linhagem validada.
