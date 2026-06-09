# 🛰️ STATION ALPHA-7 — Sistema de Monitoramento de Missão Espacial

> Global Solution 2026 — FIAP  

---

## 👥 Equipe

| Nome | RM |
|------|----|
| Sofia Petruk  | RM 570648 |

**Nome da equipe:** Missão Espacial

## Link do Vídeo 
https://youtu.be/WYCVCa8X_xc
---

## 📡 Resumo do Problema e Cenário Analisado

O cenário simulado é a **Estação Espacial ALPHA-7**, em operação no dia 2031-03-15. A estação possui seis módulos críticos monitorados em tempo real por sensores de telemetria, cujos dados são armazenados em um arquivo JSON (`dado.json`).

Durante o período analisado, a missão enfrenta duas anomalias simultâneas:

- **Falha crítica de comunicação** — a antena principal COM-01 perdeu sincronização às 07:45 UTC, e o sistema entrou em modo de contingência, operando apenas com o backup COM-02 a 34% de eficiência.
- **Alerta de pressurização** — o compartimento de armazenamento B-3 registrou queda de pressão para 94.7 kPa, abaixo do nominal.

Além disso, o nível de radiação externo atingiu **3.8 mSv/h** (limite seguro: 2.5 mSv/h), forçando o cancelamento de atividades externas (EVA). O sistema analisa essas condições, classifica a severidade de cada módulo e gera recomendações automáticas de ação.

---

## 🗂️ Estruturas de Dados Utilizadas

### Lista
Utilizada para armazenar séries temporais de leituras de geração e consumo de energia ao longo do dia.

```python
geracao_lista = []
consumo_lista = []

for leitura in dados["energia"]["leituras"]:
    geracao_lista.append(leitura["geracao"])
    consumo_lista.append(leitura["consumo"])
```

**Por quê:** listas preservam a ordem de inserção, o que é essencial para análise cronológica das leituras horárias.

---

### Fila (`deque`)
Utilizada para organizar os alertas pendentes por ordem de chegada (FIFO — primeiro a entrar, primeiro a sair).

```python
from collections import deque

fila_alertas = deque()

for evento in dados["log_eventos"]:
    if evento["tipo"] in ["ALERTA", "FALHA_SENSOR", "FALHA_CRITICA"]:
        fila_alertas.append(evento)
```

**Por quê:** a fila garante que os alertas sejam processados na ordem em que ocorreram, simulando o comportamento de sistemas reais de telemetria.

---

### Pilha
Utilizada para registrar os últimos eventos críticos analisados (LIFO — último a entrar, primeiro a sair), mantendo no máximo os 3 mais recentes.

```python
pilha = []

for evento in dados["log_eventos"]:
    if evento["tipo"] in ["FALHA_CRITICA", "FALHA_SENSOR"]:
        pilha.append(evento)
        if len(pilha) > 3:
            pilha.pop(0)
```

**Por quê:** a pilha permite consultar rapidamente o evento crítico mais recente, sem precisar percorrer o log inteiro.

---

### Dicionário (tabela hash)
Utilizado para mapear o nome de cada módulo ao seu status operacional, permitindo acesso direto por chave.

```python
status_modulos = {}

for nome, info in dados["modulos_criticos"].items():
    status_modulos[nome] = info["status"]
```

**Por quê:** dicionários oferecem acesso O(1) por chave, ideal para consultas frequentes ao estado dos módulos.

---

### Matriz (lista de listas)
Utilizada para representar as leituras de energia organizadas por horário e variável (horário, geração, consumo, reserva).

```python
matriz_energia = []

for leitura in dados["energia"]["leituras"]:
    linha = [
        leitura["horario"],
        leitura["geracao"],
        leitura["consumo"],
        leitura["reserva"]
    ]
    matriz_energia.append(linha)
```

**Por quê:** a matriz organiza dados bidimensionais (tempo × variável) de forma intuitiva e permite iteração por linha ou coluna.

---

## ⚙️ Regras Lógicas Principais do Diagnóstico

O sistema aplica três grupos de regras com operadores lógicos distintos:

### Regra 1 — Elegibilidade de Operação (`AND`)

Um módulo é considerado **apto** somente quando todas as condições são atendidas simultaneamente:

```
status = 1
E 20 ≤ geração ≤ 25 (kW)
E consumo < 20 (kW)
E reserva > 310 (kWh)
```

```python
if status == 1 and 20 <= geracao <= 25 and consumo < 20 and reserva > 310:
    # módulo elegível
```

**Por quê:** em missões espaciais, todas as condições devem estar dentro da faixa segura. Uma única falha já compromete a operação.

---

### Regra 2 — Identificação de Situações de Risco (`OR`)

A missão emite alerta caso **qualquer uma** das condições de risco esteja presente:

```
comunicação em FALHA
OU radiação acima do limite seguro (> 2.5 mSv/h)
```

```python
if comunicacao_falha or radiacao_elevada:
    print("ALERTA DE MISSÃO")
```

**Por quê:** riscos independentes devem ser detectados individualmente — basta um para acionar o alerta.

---

### Regra 3 — Verificação de Módulos Não Operacionais (`NOT`)

Um módulo é considerado **indisponível** quando não está operacional:

```python
if not modulo["status"]:
    print(f"{nome} fora de operação")
```

**Por quê:** o operador `NOT` simplifica a verificação de ausência de operacionalidade sem precisar comparar explicitamente com zero.

---

### Expressão Booleana Principal do Diagnóstico

```
missao_normal = (comunicacao OK) AND (radiacao <= 2.5) AND (reserva > 310) AND (NOT modulo_critico_falhou)
```

---

## 📈 Técnica de Previsão Utilizada e Resultado

**Técnica:** Média Móvel Simples  
**Variável analisada:** Reserva de energia (kWh)  
**Implementação:** sem bibliotecas avançadas, apenas operações básicas com listas

### Dados utilizados

| Horário (UTC) | Reserva (kWh) |
|---------------|---------------|
| 00:00 | 312.0 |
| 02:00 | 318.2 |
| 04:00 | 306.5 |
| 06:00 | 310.8 |
| 10:00 | 308.4 |
| 14:00 | 296.1 |

### Metodologia

```python
ultimas_tres = reservas[-3:]   # [310.8, 308.4, 296.1]

soma = 0
for valor in ultimas_tres:
    soma += valor

previsao = soma / len(ultimas_tres)
```

### Resultado

```
Previsão da próxima reserva: 305.10 kWh
Situação: ALERTA
Recomendação: monitorar o consumo e priorizar sistemas essenciais.
```

**Impacto na decisão:** como a previsão ficou entre 300 e 310 kWh (zona de alerta), o sistema recomendou monitoramento ativo e priorização dos sistemas essenciais antes de atingir o nível crítico.

---

## ▶️ Como Executar

### Pré-requisitos

- Python 3.10 ou superior
- Nenhuma biblioteca externa necessária

### Passos

```bash
# 1. Clone o repositório
git clone https://github.com/sofiapetruk/gs-01.git
cd gs-01

# 2. Execute o sistema
python src/sistema.py
```

> O arquivo `data/dado.json` deve estar presente na pasta `data/` na raiz do projeto.

---

## 📥 Exemplo de Entrada e Saída

### Entrada (`data/dado.json` — trecho)

```json
{
  "missao": "STATION ALPHA-7",
  "modulos_criticos": {
    "comunicacao": {
      "status": 0,
      "descricao": "FALHA — link primário offline"
    },
    "armazenamento": {
      "status": 0,
      "descricao": "ALERTA — pressurização abaixo do nominal"
    }
  },
  "energia": {
    "leituras": [
      { "horario": "2031-03-15T14:00:00Z", "geracao": 22.1, "consumo": 23.7, "reserva": 296.1 }
    ]
  }
}
```

### Saída (terminal)

```
=== ALERTAS DA MISSÃO ===

SEVERIDADE : CRITICO
MÓDULO     : comunicacao
DESCRIÇÃO  : Antena principal sem sinal — usando backup com 34% de eficiência
AÇÃO       : Ativar comunicação de backup.
--------------------------------------------------
SEVERIDADE : CRITICO
MÓDULO     : armazenamento
DESCRIÇÃO  : ALERTA — pressurização abaixo do nominal
AÇÃO       : Realizar diagnóstico imediato do módulo.
--------------------------------------------------
SEVERIDADE : ALERTA
MÓDULO     : energia
DESCRIÇÃO  : Reserva baixa: 296.1 kWh
AÇÃO       : Monitorar consumo energético.
--------------------------------------------------
SEVERIDADE : ALERTA
MÓDULO     : energia
DESCRIÇÃO  : Consumo elevado: 23.7 kW
AÇÃO       : Reavaliar atividades de alto consumo.
--------------------------------------------------
SEVERIDADE : ALERTA
MÓDULO     : radiacao
DESCRIÇÃO  : Radiação em 3.8 mSv/h
AÇÃO       : Suspender atividades externas (EVA).
--------------------------------------------------

=== ANÁLISE E PREVISÃO DE ENERGIA ===

Previsão da próxima reserva: 305.10 kWh

Situação: ALERTA
Recomendação: monitorar o consumo e priorizar sistemas essenciais.
```

---

## 🚨 Recomendações Geradas pelo Sistema

O sistema classifica automaticamente os alertas em três níveis de severidade e gera uma recomendação de ação para cada um:

| Severidade | Módulo | Situação detectada | Recomendação |
|------------|--------|--------------------|--------------|
| 🔴 CRÍTICO | comunicacao | Antena principal offline, backup a 34% | Ativar comunicação de backup |
| 🔴 CRÍTICO | armazenamento | Pressurização do compartimento B-3 em 94.7 kPa | Realizar diagnóstico imediato do módulo |
| 🟡 ALERTA | energia | Reserva em 296.1 kWh (abaixo de 310) | Monitorar consumo energético |
| 🟡 ALERTA | energia | Consumo de 23.7 kW acima do limite | Reavaliar atividades de alto consumo |
| 🟡 ALERTA | radiacao | Radiação em 3.8 mSv/h (limite: 2.5) | Suspender atividades externas (EVA) |

Os alertas são ordenados por prioridade (CRÍTICO antes de ALERTA) antes de serem exibidos.

---

## 🎥 Vídeo de Apresentação

> 📺 [Link do vídeo no YouTube — a ser adicionado]

---

## 📁 Estrutura do Repositório

```
gs-01/
├── data/
│   └── dado.json   # Dados de telemetria da missão
├── docs/
│   ├── relatorio.pdf              # Relatório da entrega
│   └── uso_da_ia.md               # Declaração de uso de IA
├── src/
│   └── sistema.ipynb              # Notebook principal do sistema
└── README.md
```

---

## 🧠 Conclusões e Aprendizados

O desenvolvimento deste projeto proporcionou a aplicação prática dos conceitos de Python estudados durante o semestre em um cenário realista e desafiador.

**Estruturas de dados na prática:** entender quando usar lista, fila, pilha ou dicionário fez diferença direta na legibilidade e na eficiência do código. A escolha da estrutura certa simplificou soluções que seriam complexas de outra forma.

**Lógica booleana aplicada:** os operadores `AND`, `OR` e `NOT` deixaram de ser conceitos abstratos e passaram a representar decisões reais do sistema — como determinar se uma missão está segura ou se um alerta deve ser disparado.

**Análise de dados sem bibliotecas:** implementar a média móvel apenas com listas e loops mostrou que é possível construir lógica analítica útil com recursos básicos da linguagem, sem depender de ferramentas externas.

**Leitura de JSON:** percorrer estruturas aninhadas de dicionários e listas em Python foi um dos aprendizados mais práticos, pois reflete exatamente como dados reais chegam de APIs e sensores no mundo profissional.

**Orientação a objetos:** a classe `MonitorMissao` organizou o sistema de forma que cada responsabilidade ficou em seu próprio método, tornando o código mais fácil de entender, testar e expandir.
