# Rota Mais Rápida — Distrito Federal

Número da Lista: Grafos<br>
Conteúdo da Disciplina: Grafos — Caminho Mínimo (Dijkstra)<br>

## Alunos

| Matrícula | Aluno                       |
| --------- | --------------------------- |
| 222006276 | Luís Gustavo Lopes Oliveira |
| 222006211 | Vitor Valerio Hoffmann      |

## Sobre

Aplicação web que calcula a **rota mais rápida entre dois pontos do Distrito
Federal**, usando a malha viária real do OpenStreetMap. O usuário clica na
origem e no destino sobre o mapa e recebe o caminho de menor tempo, junto com
as métricas de execução do algoritmo.

**Como o problema foi modelado.** A malha viária vira um **grafo dirigido e
ponderado**: cruzamentos são os vértices (guardando latitude e longitude) e
trechos de rua são as arestas. O peso de cada aresta é o **tempo de percurso**,
não a distância:

```
tempo (s) = comprimento do trecho (m) ÷ velocidade da via (m/s)
```

Assim o Dijkstra prefere um desvio maior por via rápida a um caminho curto por
ruas lentas. Como o grafo é dirigido, ruas de mão única são respeitadas — 17%
das arestas existem em um sentido só.

O grafo do DF tem **85.584 vértices e 200.373 arestas**, cobrindo todas as
regiões administrativas. O caminho mínimo é encontrado por uma implementação
própria do **Dijkstra** (`backend/dijkstra.py`), com fila de prioridade e
reconstrução do caminho pelos predecessores.

## Screenshots

**1. Tela inicial** — o mapa do DF esperando o primeiro clique, com o tamanho do
grafo carregado no rodapé do painel.

![Tela inicial com o mapa do Distrito Federal](assets/01-mapa-inicial.jpg)

**2. Rota longa: Gama → Planaltina** — 59,8 min de viagem estimada. O Dijkstra
visitou **81.921 dos 85.584 vértices** (235,6 ms) para achar um caminho de 323
cruzamentos: quase o grafo inteiro, porque o algoritmo se espalha em todas as
direções sem saber onde fica o destino.

![Rota de Gama até Planaltina traçada no mapa](assets/02-rota-gama-planaltina.jpg)

**3. Rota curta: Asa Norte → Asa Sul** — 9,2 min, 71 cruzamentos. Aqui bastaram
**7.244 nós visitados** e 29,7 ms, mais de dez vezes menos esforço que a rota
anterior.

![Rota da Asa Norte até a Asa Sul traçada no mapa](assets/03-rota-asa-norte-asa-sul.jpg)

**4. Documentação automática da API** — os endpoints `POST /rota` e
`GET /status` em `http://127.0.0.1:8000/docs`.

![Swagger UI da API mostrando os endpoints /rota e /status](assets/04-api-docs.jpg)

## Instalação

Linguagem: Python 3.9+ e JavaScript<br>
Framework: FastAPI (backend) e React + Vite com Leaflet (frontend)<br>

**Pré-requisitos:** Python 3.9 ou superior e Node.js 18 ou superior.

```bash
# 1. ambiente virtual (evita o erro externally-managed-environment)
python3 -m venv venv
source venv/bin/activate      # no Windows: venv\Scripts\activate

# 2. dependências do backend
pip install -r backend/requirements.txt

# 3. dependências do frontend (só na primeira vez)
cd frontend && npm install && cd ..

# 4. sobe backend e frontend juntos (Ctrl+C para parar os dois)
python run.py
```

Abra **[http://127.0.0.1:5500](http://127.0.0.1:5500)**. A API fica em
`http://127.0.0.1:8000`, com documentação automática em `/docs`.

> **Primeira execução:** o download da malha do DF leva cerca de 1 minuto. A
> interface mostra "Aguardando a API…" e se conecta sozinha quando o backend
> fica pronto — não precisa recarregar. Depois o grafo fica em cache
> (`backend/grafos/`, ~8 MB) e carrega em ~1 segundo.

## Uso

1. **Clique na origem** em qualquer ponto do mapa — marcador **verde**.
2. **Clique no destino** — marcador **vermelho**, e o cálculo começa na hora.
3. **A rota aparece em azul** e o mapa se reenquadra para mostrá-la inteira.
4. **Para testar outra**, é só clicar de novo: o terceiro clique começa uma
   busca nova. O botão **Limpar** também zera tudo.

Não precisa clicar em cima de uma rua — o sistema acha o cruzamento mais
próximo do ponto clicado.

### As métricas

| Métrica             | O que é                                                     |
| ------------------- | ----------------------------------------------------------- |
| Tempo estimado      | Soma dos pesos do caminho — a viagem prevista pelo modelo.  |
| Nós visitados       | Vértices que o Dijkstra tirou da fila. Mede o esforço dele. |
| Cálculo do Dijkstra | Tempo só do algoritmo, sem rede.                            |

"Nós visitados" cresce muito com a distância: a rota da Asa Norte para a Asa Sul
visita 7 mil nós (30 ms), enquanto Gama → Planaltina visita **82 mil dos 85 mil
vértices** (236 ms). Isso mostra na prática o Dijkstra explorando em todas as
direções, sem saber onde fica o destino, a deixa natural para comparar com A\*.

## Outros

### A API

**`POST /rota`** — recebe origem e destino, devolve o caminho mínimo:

```bash
curl -X POST http://127.0.0.1:8000/rota -H "Content-Type: application/json" \
  -d '{"origem_lat":-16.0192,"origem_lon":-48.0642,
       "destino_lat":-15.7942,"destino_lon":-47.8822}'
```

```json
{
  "caminho_coordenadas": [[-16.0188, -48.0644], "..."],
  "tempo_estimado_segundos": 1717.7,
  "nos_visitados": 44929,
  "tempo_calculo_ms": 161.7
}
```

Devolve **404** quando não há caminho entre os pontos.

**`GET /status`** — região carregada e tamanho do grafo:

```json
{ "regiao": "Distrito Federal, Brazil", "nos": 85584, "arestas": 200373 }
```

### Estrutura do projeto

```
backend/
├── graph.py           estrutura de grafo, haversine e índice espacial
├── dijkstra.py        algoritmo de caminho mínimo
├── graph_loader.py    download do OpenStreetMap, conversão e cache
└── main.py            API FastAPI
frontend/src/
├── App.jsx            estado da aplicação
├── api.js             chamadas à API
└── components/        Mapa.jsx, PainelRota.jsx, CardResultado.jsx
run.py                 sobe backend e frontend juntos
```

### Implementação

- **`graph.py`** — a classe `Grafo` em lista de adjacência, a distância
  `haversine` e o `IndiceEspacial`, uma grade que acha o cruzamento mais
  próximo de um clique visitando anéis de células ao redor dele. Sem ele cada
  clique compararia o ponto com os 85 mil vértices: são 67 ms de varredura
  linear contra 0,91 ms com a grade, duas vezes por requisição.
- **`dijkstra.py`** — o algoritmo completo, com fila de prioridade e
  reconstrução do caminho pelos predecessores. A única estrutura de apoio é o
  `heapq` da biblioteca padrão.

**Bibliotecas**: o **OSMnx** baixa os dados geográficos do OpenStreetMap; eles
são convertidos para a nossa classe `Grafo` e **a busca nunca roda sobre o grafo
do `networkx`**. **FastAPI** serve a API e **React/Leaflet** desenham a
interface.

### Trocando a região

Por padrão carrega o DF inteiro. Para rodar sobre uma região menor:

```bash
REGIAO_OSM="Gama, Brasilia, Brazil" python run.py
```

Cada região tem seu cache próprio, então dá para alternar sem baixar de novo.
O Gama sozinho tem 5.372 vértices — cerca de 1/16 do DF.

### Rodando as partes separadas

```bash
python backend/main.py             # só a API
cd frontend && npm run dev         # só a interface
cd backend && python dijkstra.py   # só o algoritmo, num grafo de teste
```

O `dijkstra.py` roda sobre um grafo de 4 vértices, sem OSMnx nem rede — bom
para verificar o algoritmo isolado.

## Vídeo de Apresentação

[Assista no YouTube](https://youtu.be/FgOLd6GFR54)
