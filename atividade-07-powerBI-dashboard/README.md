# ✈ Análise de Voos Domésticos — Índia
 
**Dashboard Power BI | Atividade 06 — APEX 2º Bimestre**
**Modelagem e Ferramentas de Análise de Dados**
 
> *Preços, rotas e companhias aéreas — 300.153 voos analisados*
 
---
 
## 👥 Equipe — CTRL ALT DELAS
 
- **Maria Eduarda de Carvalho Cortellini** — RA: 52420167
---
 
## 📋 Sobre o Projeto
 
Este projeto consiste em um dashboard interativo construído no **Power BI Desktop** para análise de voos domésticos na Índia. O dataset contém **300.153 registros** de voos operados por 6 companhias aéreas, conectando 6 grandes cidades indianas em duas classes de serviço (Economy e Business).
 
O objetivo do dashboard é responder a perguntas analíticas como:
 
- Qual companhia aérea cobra os preços mais altos?
- Comprar passagens com antecedência realmente vale a pena?
- Como o horário de partida influencia o preço das passagens?
- Quais são as rotas mais movimentadas?
- Qual a diferença média de preço entre Business e Economy?
---
 
## 📊 Dataset
 
**Arquivo fonte:** `Clean_Dataset.csv`
 
| Característica | Valor |
|---|---|
| Total de registros | 300.153 voos |
| Companhias aéreas | 6 (Vistara, Air_India, Indigo, GO_FIRST, AirAsia, SpiceJet) |
| Cidades atendidas | 6 (Delhi, Mumbai, Bangalore, Kolkata, Hyderabad, Chennai) |
| Classes | Economy, Business |
| Períodos do dia | 6 (Early_Morning, Morning, Afternoon, Evening, Night, Late_Night) |
| Tipos de parada | zero, one, two_or_more |
| Faixa de preço | ₹ 1.105 a ₹ 123.071 |
| Antecedência | 1 a 49 dias |
 
---
 
## 🌟 Modelagem Dimensional (Star Schema)
 
O modelo foi construído seguindo o conceito de **esquema estrela** apresentado na Aula 07, com uma tabela fato central conectada a cinco tabelas de dimensão.
 
### Tabela Fato
 
| Tabela | Descrição |
|---|---|
| `fVoos` | Registros de voos com medidas de preço, duração e antecedência |
 
### Tabelas de Dimensão
 
| Tabela | Conteúdo |
|---|---|
| `dCompanhia` | 6 companhias aéreas |
| `dCidade` | 6 cidades (compartilhada entre origem e destino) |
| `dClasse` | Economy / Business |
| `dHorario` | 6 períodos do dia com coluna de ordenação |
| `dParadas` | zero / one / two_or_more com coluna de ordenação |
 
### Tabela Auxiliar
 
| Tabela | Função |
|---|---|
| `_Medidas` | Tabela vazia que organiza todas as medidas DAX em um único local |
 
---
 
## 🔗 Relacionamentos
 
| De (fVoos) | Para | Cardinalidade | Status |
|---|---|---|---|
| `Companhia` | `dCompanhia[Companhia]` | N:1 | ✅ Ativo |
| `Cidade Origem` | `dCidade[Cidade]` | N:1 | ✅ Ativo |
| `Cidade Destino` | `dCidade[Cidade]` | N:1 | ⚠️ Inativo |
| `Classe` | `dClasse[Classe]` | N:1 | ✅ Ativo |
| `Horário Partida` | `dHorario[Horário]` | N:1 | ✅ Ativo |
| `Horário Chegada` | `dHorario[Horário]` | N:1 | ⚠️ Inativo |
| `Paradas` | `dParadas[Paradas]` | N:1 | ✅ Ativo |
 
> 💡 Os relacionamentos inativos (Cidade Destino e Horário Chegada) são ativados sob demanda via função `USERELATIONSHIP` nas medidas DAX, permitindo análises por destino sem duplicar dimensões.
 
---
 
## 📂 Hierarquia Geográfica
 
Foi criada uma hierarquia na tabela `fVoos` permitindo drill-down geográfico:
 
```
📂 Hierarquia Geográfica
   ├─ Cidade Origem
   └─ Cidade Destino
```
 
Esta hierarquia possibilita análises OLAP (drill-down e roll-up) diretamente nos visuais do dashboard.
 
---
 
## 🛠 Linguagem M (Power Query)
 
### Consulta `fVoos` (Tabela Fato)
 
```m
let
    Fonte = Csv.Document(
        File.Contents("C:\Users\52420167\Downloads\Clean_Dataset.csv"),
        [Delimiter=",", Columns=12, Encoding=65001, QuoteStyle=QuoteStyle.None]
    ),
    #"Cabeçalhos Promovidos" = Table.PromoteHeaders(Fonte, [PromoteAllScalars=true]),
    #"Coluna Índice Removida" = Table.RemoveColumns(#"Cabeçalhos Promovidos",{""}),
    #"Tipo Alterado" = Table.TransformColumnTypes(#"Coluna Índice Removida",{
        {"airline", type text}, {"flight", type text},
        {"source_city", type text}, {"departure_time", type text},
        {"stops", type text}, {"arrival_time", type text},
        {"destination_city", type text}, {"class", type text},
        {"duration", type number}, {"days_left", Int64.Type},
        {"price", Int64.Type}
    }),
    #"Faixa Antecedência" = Table.AddColumn(#"Tipo Alterado", "faixa_antecedencia",
        each if [days_left] <= 7 then "1. Última Semana (1-7d)"
        else if [days_left] <= 15 then "2. Curto Prazo (8-15d)"
        else if [days_left] <= 30 then "3. Médio Prazo (16-30d)"
        else "4. Planejado (31-49d)", type text),
    #"Faixa Duração" = Table.AddColumn(#"Faixa Antecedência", "faixa_duracao",
        each if [duration] <= 3 then "Curta (até 3h)"
        else if [duration] <= 10 then "Média (3-10h)"
        else "Longa (10h+)", type text),
    #"Rota" = Table.AddColumn(#"Faixa Duração", "rota",
        each [source_city] & " → " & [destination_city], type text),
    #"Colunas Renomeadas" = Table.RenameColumns(#"Rota",{
        {"airline","Companhia"}, {"flight","Voo"},
        {"source_city","Cidade Origem"}, {"departure_time","Horário Partida"},
        {"stops","Paradas"}, {"arrival_time","Horário Chegada"},
        {"destination_city","Cidade Destino"}, {"class","Classe"},
        {"duration","Duração (h)"}, {"days_left","Dias Antecedência"},
        {"price","Preço (₹)"}, {"faixa_antecedencia","Faixa Antecedência"},
        {"faixa_duracao","Faixa Duração"}, {"rota","Rota"}
    })
in
    #"Colunas Renomeadas"
```
 
### Transformações aplicadas
 
1. **Importação do CSV** com encoding UTF-8
2. **Remoção da coluna índice** vinda do CSV
3. **Tipagem correta** das colunas (texto, número decimal, inteiro)
4. **Criação de coluna calculada `Faixa Antecedência`** (coluna condicional em 4 faixas)
5. **Criação de coluna calculada `Faixa Duração`** (curta, média, longa)
6. **Criação de coluna calculada `Rota`** (concatenação Origem → Destino)
7. **Renomeação de todas as colunas** para português
### Consulta `dCidade` (dimensão compartilhada)
 
```m
let
    Origens = Table.SelectColumns(fVoos,{"Cidade Origem"}),
    Destinos = Table.SelectColumns(fVoos,{"Cidade Destino"}),
    OrigensRenomeada = Table.RenameColumns(Origens,{{"Cidade Origem","Cidade"}}),
    DestinosRenomeada = Table.RenameColumns(Destinos,{{"Cidade Destino","Cidade"}}),
    União = Table.Combine({OrigensRenomeada, DestinosRenomeada}),
    Distintas = Table.Distinct(União),
    Ordenado = Table.Sort(Distintas,{{"Cidade", Order.Ascending}}),
    Índice = Table.AddIndexColumn(Ordenado, "ID Cidade", 1, 1, Int64.Type)
in
    Índice
```
 
### Consulta `dHorario` (dimensão com ordenação customizada)
 
```m
let
    Fonte = #table(
        {"Horário","Ordem"},
        {
            {"Early_Morning",1},{"Morning",2},{"Afternoon",3},
            {"Evening",4},{"Night",5},{"Late_Night",6}
        }
    ),
    Tipo = Table.TransformColumnTypes(Fonte,{{"Horário",type text},{"Ordem",Int64.Type}}),
    Índice = Table.AddIndexColumn(Tipo, "ID Horário", 1, 1, Int64.Type)
in
    Índice
```
 
> As demais dimensões (`dCompanhia`, `dClasse`, `dParadas`) seguem padrão semelhante, com `Table.Distinct` e `Table.AddIndexColumn` para gerar chaves.
 
---
 
## 📐 Medidas DAX
 
Todas as medidas estão organizadas na tabela `_Medidas`.
 
### Medidas Essenciais
 
```dax
Total de Voos = COUNTROWS(fVoos)
 
Preço Médio = ROUND(AVERAGE(fVoos[Preço (₹)]), 0)
 
Preço Máximo = MAX(fVoos[Preço (₹)])
 
Preço Mínimo = MIN(fVoos[Preço (₹)])
 
Duração Média (h) = ROUND(AVERAGE(fVoos[Duração (h)]), 2)
```
 
### Medidas Intermediárias
 
```dax
Receita Total = SUM(fVoos[Preço (₹)])
 
Companhias Distintas = DISTINCTCOUNT(fVoos[Companhia])
 
Rotas Distintas = DISTINCTCOUNT(fVoos[Rota])
```
 
### Medidas Avançadas (CALCULATE + VAR)
 
```dax
% Voos Business =
VAR Total = COUNTROWS(fVoos)
VAR Business =
    CALCULATE(
        COUNTROWS(fVoos),
        dClasse[Classe] = "Business"
    )
RETURN
    DIVIDE(Business, Total, 0)
```
 
```dax
Preço Médio Business =
CALCULATE(
    [Preço Médio],
    dClasse[Classe] = "Business"
)
 
Preço Médio Economy =
CALCULATE(
    [Preço Médio],
    dClasse[Classe] = "Economy"
)
 
Diferença Business vs Economy =
[Preço Médio Business] - [Preço Médio Economy]
```
 
### Medida com USERELATIONSHIP
 
Esta medida demonstra o uso do relacionamento inativo, ativando dinamicamente a conexão entre `fVoos[Cidade Destino]` e `dCidade[Cidade]`:
 
```dax
Preço Médio por Destino =
CALCULATE(
    [Preço Médio],
    USERELATIONSHIP(fVoos[Cidade Destino], dCidade[Cidade])
)
```
 
---
 
## 🎨 Visuais do Dashboard
 
O dashboard é composto por **uma única página** com layout otimizado para visualização em tela cheia.
 
### Cabeçalho
- **Título principal:** "Análise de Voos Domésticos - Índia" (28pt, bold)
- **Subtítulo:** "Preços, rotas e companhias aéreas - 300.153 voos analisados" (16pt, itálico)
- **Imagem decorativa:** ilustração temática de aviação
### Cartões KPI (4 cartões — topo)
 
| KPI | Medida | Valor |
|---|---|---|
| Voos analisados | `Total de Voos` | 300 Mil |
| Preço médio | `Preço Médio` | ₹ 20,89 Mil |
| Duração média (horas) | `Duração Média (h)` | 1,03 Mil |
| Companhias aéreas | `Companhias Distintas` | 6 |
 
### Gráficos Principais (4 gráficos)
 
| Visual | Tipo | Dados | Pergunta respondida |
|---|---|---|---|
| Qual companhia cobra mais caro? | Barras agrupadas horizontais | `dCompanhia[Companhia]` × `Preço Médio` | Comparação de preço entre cias |
| Comprar com antecedência vale a pena? | Linha | `Dias Antecedência` × `Preço Médio` | Variação do preço por antecedência |
| Preço por horário: Business x Economy | Colunas empilhadas 100% | `dHorario[Horário]` × `Preço Médio` × `Classe` | Comparação de classes por horário |
| De onde mais saem voos? | Barras com hierarquia | `Hierarquia Geográfica` × `Total de Voos` | Volume de voos por origem (com drill-down) |
 
### Segmentações de Dados (3 slicers)
 
| Slicer | Campo | Tipo |
|---|---|---|
| Filtrar por rota | `fVoos[Rota]` | Lista |
| Filtrar por origem | `Hierarquia Geográfica.Cidade Origem` | Lista |
| Faixa de preço (₹) | `fVoos[Preço (₹)]` | Intervalo numérico |
 
---
 
## ❓ Perguntas Analíticas e Respostas
 
### 1. Qual companhia cobra mais caro?
**Vistara** lidera com preços médios significativamente acima da concorrência, seguida pela **Air_India**. As companhias low-cost (AirAsia, Indigo, GO_FIRST) mantêm preços médios mais baixos.
 
### 2. Comprar com antecedência vale a pena?
**Sim, e muito.** O gráfico de linha mostra uma queda acentuada do preço médio à medida que `Dias Antecedência` aumenta de 1 para ~10 dias. Após 15 dias de antecedência, o preço se estabiliza em torno de ₹ 20.000.
 
### 3. Quem voa mais saindo de onde?
**Delhi e Mumbai** dominam como cidades de origem, refletindo seu papel como principais hubs aéreos da Índia.
 
### 4. Diferença entre Business e Economy?
Voos Business custam, em média, **muito mais** do que Economy, mantendo essa proporção consistente em todos os horários do dia. Os gráficos empilhados em 100% mostram que a classe Business representa uma fração relativamente pequena dos voos, mas com preços desproporcionalmente altos.
 
---
 
## 🔧 Tecnologias Utilizadas
 
- **Microsoft Power BI Desktop**
- **Power Query (Linguagem M)** — ETL e modelagem de dados
- **DAX (Data Analysis Expressions)** — medidas e cálculos
- **Star Schema** — modelagem dimensional clássica
---
 
## 📁 Estrutura de Arquivos
 
```
📁 Projeto
   ├─ 📄 Clean_Dataset.csv          (dataset fonte)
   ├─ 📄 ANALISE_DE_VOOS_DOMESTICOS_INDIA_CTRL_ALT_DELAS.pbix  (dashboard)
   └─ 📄 README.md                  (este arquivo)
```
 
---
 
## 🎓 Conceitos Aplicados
 
Este projeto consolida os principais conceitos abordados ao longo do bimestre:
 
- ✅ **Fundamentos de BI** (Aula 02)
- ✅ **Power BI Desktop — interface e navegação** (Aula 03)
- ✅ **ETL com Power Query** (Aula 04 e 05)
- ✅ **Linguagem M** (Aula 05 e 10) — colunas condicionais, transformações
- ✅ **OLAP, drill-down e modelagem dimensional** (Aula 07)
- ✅ **DAX — medidas, CALCULATE, USERELATIONSHIP, VAR** (Aula 10)
- ✅ **Construção de dashboards profissionais** (Aula 12)
---
 
*Trabalho desenvolvido para a disciplina de Modelagem e Ferramentas de Análise de Dados — Prof. Alan Gonçalves — UniFAJ/UniMAX*
 
