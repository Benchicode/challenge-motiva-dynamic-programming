# Escadas Rolantes: Manutenção Orientada por Dados

**Dynamic Programming · Challenge Motiva + FIAP 2026/2027**

Priorização da manutenção das 197 escadas rolantes da Linha 4-Amarela (ViaQuatro / Motiva) com programação dinâmica e algoritmos de caminhos em grafos.

## Integrantes

| Nome | RM |
|---|---|
| - André Rosa Ferreira Filho -  | RM:567149
| - Cainã Sandes Batista -  | RM:568571
| - Michel Lima Benchimol -  | RM:567345
| - Yuri Aroste Fukamachi - | RM:567314 

## Estrutura do repositório

```
├── notebook.ipynb          # notebook executado com todo o código e as interpretações
├── README.md
└── dados/
    ├── ecr_09-26.XLSX      # dataset da Motiva (notas de manutenção das escadas)
    ├── estacoes.csv        # vértices do grafo: estações + base, com coordenadas
    └── trechos.csv         # arestas do grafo: origem, destino, minutos, modo
```

## Como executar

```
pip install pandas openpyxl jupyter
jupyter notebook notebook.ipynb
```

Execute todas as células em ordem (*Run All*). A leitura do Excel leva alguns segundos.

## Bibliotecas usadas

- `pandas`: leitura do Excel e dos CSVs, preparação dos dados e tabelas de resultado.
- `collections.deque`: fila da BFS.
- `time`: medição do tempo de execução.

---

## 1. Dados

O dataset da Motiva traz **notas de manutenção do SAP**, e não leituras contínuas de sensores. Cada nota é uma ocorrência em uma escada (alarme, falha, inspeção), com data, sintoma e indicação de indisponibilidade.

**Preparação:**
1. Remoção das notas canceladas (status com `CANC`).
2. Extração da estação e da escada a partir do local de instalação (`SP04-6-ECR-PIN-ER01` → estação `PIN`, escada `PIN-ER01`).
3. Período a partir de 2024: 27.389 notas, **197 escadas** e 12 estações.

**Peso de cada leitura (nota):**

| Situação | Peso |
|---|---|
| Toda ocorrência | +1 |
| Classificada como falha | +2 |
| Escada indisponível | +3 |
| Sintoma ligado aos sinais da Motiva (vibração, temperatura, corrente/tensão, óleo, velocidade) | +2 ou +3 |

## 2. Degradação acumulada (programação dinâmica)

```
D(i) = D(i - 1) × 0,99 + peso(i)        com D(-1) = 0
```

O fator 0,99 faz as ocorrências antigas pesarem menos que as recentes. A função calcula a **curva** de degradação de cada escada, em três versões:

| Versão | Como funciona | Chamadas/passos para n leituras |
|---|---|---|
| `curva_recursiva` | recursão pura, refaz tudo a cada ponto | ≈ n²/2 |
| `curva_memoizada` | recursão + dicionário `memo` | n |
| `curva_tabulada` | lista preenchida de baixo para cima | n |

**Resultados:**
- As três versões geram valores idênticos nos 27.389 pontos.
- A recursiva pura foi cerca de **70 vezes mais lenta** que a tabulação no conjunto completo.
- Indo de 50 para 400 leituras (8×), o tempo da recursiva cresce cerca de 64× (quadrático), enquanto memoizada e tabulação crescem cerca de 8× (linear).
- A tabulação foi a escolhida para o ranking: é a mais rápida e não esbarra no limite de recursão do Python (cerca de 1000 chamadas aninhadas).

**Ranking de criticidade (top 5):**

| # | Escada | Estação | Leituras | Degradação |
|---|---|---|---|---|
| 1 | PTA-ER04 | Paulista | 230 | 411,0 |
| 2 | PIN-ER05 | Pinheiros | 359 | 385,8 |
| 3 | LUZ-ER04 | Luz | 341 | 381,9 |
| 4 | PIN-ER32 | Pinheiros | 247 | 379,1 |
| 5 | PIN-ER02 | Pinheiros | 222 | 376,1 |

Pinheiros concentra 7 das 15 escadas mais críticas e tem a maior degradação média por estação. A escada com mais notas (PIN-ER24, 447) fica só em 6º lugar, o que mostra que a gravidade e a data das ocorrências pesam mais que a quantidade.

## 3. Rede de estações (grafo)

- **13 vértices:** 12 estações da Linha 4 e a base de manutenção (Pátio Vila Sônia).
- **17 arestas** com peso em minutos: metrô entre estações vizinhas, ligações a pé e atalhos de van pelas ruas.
- Representação em **lista de adjacência** (dicionário de listas). Toda escada do dataset está ligada a uma estação.

| Algoritmo | Objetivo | Exemplo: Base → Luz |
|---|---|---|
| BFS | menos trechos | 3 trechos, 60 min (van) |
| Dijkstra | menor tempo | 11 trechos, 29 min (metrô) |

As rotas são reconstruídas pelo dicionário `anterior`. Menos trechos não significa menos tempo: para deslocar a equipe, o Dijkstra é o critério correto.

## 4. A* e comparação com Dijkstra

**Heurística:** `h(n) = distância em linha reta até o destino / velocidade máxima da rede`.

Ela é **admissível** porque nenhum caminho é mais curto que a linha reta e nenhum trecho é mais rápido que a velocidade máxima (0,74 km/min). Portanto `h(n)` nunca superestima o tempo real, e o A* garante o caminho ótimo.

**Resultados:**
- Mesmo tempo ótimo que o Dijkstra nos **156 pares** de origem e destino.
- Cerca de **22% menos estações expandidas** no total. Exemplo: Faria Lima → Luz, Dijkstra expande 10 estações e A* expande 6.
- Partindo da base (ponta da linha), não há ganho, porque não existe "lado errado" para evitar.
- O tempo de execução é parecido nesta rede pequena. O ganho do A* cresce em redes maiores.

## 5. Plano de manutenção: `planejar_turno(base, horas)`

Percorre o ranking da escada mais crítica para a menos crítica. Uma escada entra no plano só se `deslocamento + 60 min de atendimento + volta à base` couber no tempo restante do turno. Todas as rotas são de menor tempo (Dijkstra).

| Turno | Escadas atendidas | Tempo usado |
|---|---|---|
| 8 h | 6 (top 6 do ranking) | 434 min |
| 6 h | 4 (top 4 do ranking) | 314 min |

**Limitação:** seguir a prioridade de forma estrita causa vai e vem entre estações (Pinheiros → Luz → Pinheiros). Uma melhoria possível é agrupar escadas da mesma estação quando a criticidade delas for próxima.

## Premissas e limitações

- Os pesos das notas e o fator 0,99 são premissas do grupo, a serem validadas com a Motiva.
- Os tempos dos trechos e as coordenadas das estações são aproximados.
- O tempo de atendimento por escada é fixo em 60 minutos.
- O dataset não tem fluxo de passageiros. A relação entre estações de integração e desgaste é uma hipótese, não uma conclusão dos dados.
