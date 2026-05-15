# 🤖 Motor de Busca Semântico — Desafio Técnico FGV IBRE 

Pipeline de busca semântica aplicado a notícias econômicas brasileiras, construído com `sentence-transformers`.

---

## Como rodar

O projeto foi desenvolvido no Google Colab e pode ser reproduzido com um clique:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1PyRYq3K78c2QAxKm-JEEnPJqrprVvPcA?usp=sharing)


### Bibliotecas utilizadas
```
sentence-transformers
beautifulsoup4
numpy
scikit-learn
```

### Estrutura do repositório
```
├── README.md
├── notebook.ipynb          
├── dados/
│   └── noticias_brutas.json 
└── dados_limpos.json        
```

## Decisões técnicas

### Etapa 1 — Limpeza de texto

A inspeção dos dados revelou seis tipos de sujeira no campo `texto`:

Tags HTML
Entidades HTML
Timestamp no cabeçalho
Data vazada no corpo do texto
Quebras de linha múltiplas
Espaços excessivos

**Tratamento de casos extremos:** a notícia de ID 18 resultou em apenas 6 caracteres após a limpeza (`"Selic."`), indicando que o conteúdo original era essencialmente vazio. Em vez de removê-la do dataset, ela foi mantida com o campo `"valido": false`, preservando a rastreabilidade e documentando a decisão de forma transparente. As etapas seguintes do pipeline utilizam apenas registros com `"valido": true` (19 de 20 notícias).

---

### Etapa 2 — Geração de Embeddings

**Modelo escolhido:** `paraphrase-multilingual-MiniLM-L12-v2`

Para justificar a escolha, foram testados três modelos com as mesmas queries de validação:
MiniLM-L12-v2, bert-base-portuguese-cased  e o mpnet-base-v2

O MiniLM se mostrou o mais equilibrado: acertou as três queries com scores consistentes (0.59–0.65) e ainda tem a vantagem de ser significativamente mais leve (~120MB vs ~420MB do mpnet), o que é relevante em cenários de produção.

O bert-base-portuguese-cased foi descartado por ser um modelo base sem fine-tuning para tarefas de similaridade semântica, o que explica os scores mais baixos.

Cada notícia foi representada como `"{titulo}. {texto}"`. O título carrega informação semântica densa e complementa o corpo do texto, resultando em embeddings mais representativos do conteúdo da notícia como um todo.

---

### Etapa 3 — Motor de Busca Semântico

A busca funciona em três passos:

1. A query do usuário é transformada em embedding pelo mesmo modelo da Etapa 2
2. A similaridade de cosseno é calculada entre o embedding da query e todos os embeddings do corpus
3. As notícias são ordenadas por score decrescente e os `top_k` resultados são retornados

A **similaridade de cosseno** foi escolhida por ser a métrica padrão para comparação de embeddings de texto.

---

## Avaliação dos resultados

Resultados das três queries de validação com o modelo MiniLM:

**"mudanças na taxa de juros"**
| # | Score | Título |
|---|---|---|
| 1 | 0.6206 | Copom mantém Selic em 13,75% ao ano pela quarta reunião consecutiva |
| 2 | 0.6053 | Crédito total no Brasil atinge R$ 5,6 trilhões com desaceleração no crescimento |
| 3 | 0.5976 | Selic deve recuar a 9% até o fim de 2024, projetam economistas |

Os três resultados são relevantes, o modelo capturou essa relação semântica corretamente.

**"mercado de trabalho e desemprego"**
| # | Score | Título |
|---|---|---|
| 1 | 0.6545 | Desemprego juvenil no Brasil ainda preocupa apesar de melhora geral |
| 2 | 0.6180 | Taxa de desemprego cai para 7,9% no segundo trimestre, menor nível desde 2014 |
| 3 | 0.4598 | Setor de serviços cresce 0,6% em junho e supera expectativas |

#1 e #2 são precisos. O #3 tem score mais baixo (0.46) e relevância apenas indireta, pois serviços é um setor intensivo em mão de obra, o que pode explicar a associação semântica.

**"inflação e preços ao consumidor"**
| # | Score | Título |
|---|---|---|
| 1 | 0.6271 | Inflação ao produtor (IPA) desacelera e pressão sobre preços finais diminui |
| 2 | 0.5882 | IGP-M registra terceira deflação consecutiva em agosto |
| 3 | 0.5812 | Selic deve recuar a 9% até o fim de 2024, projetam economistas |

#1 e #2 são altamente relevantes. O #3 (Selic) aparece porque a taxa de juros é o principal instrumento de controle da inflação.

## Conclusão ✨

Os testes indicam que o modelo MiniLM apresentou desempenho consistente na busca semântica de notícias econômicas. Em todas as queries avaliadas, as notícias mais relevantes foram corretamente posicionadas nas primeiras colocações, enquanto resultados de relevância indireta receberam scores menores, preservando a qualidade do ranking. Além da correspondência textual, o modelo demonstrou capacidade de capturar relações econômicas implícitas, como a conexão entre juros, crédito, inflação e emprego.

---

## Limitações e possíveis melhorias

- Com apenas 19 notícias válidas, os resultados são difíceis de generalizar - Um corpus maior permitiria uma avaliação mais robusta. 
- O modelo MiniLM foi treinado em dados multilíngues - Um modelo com fine-tune em textos econômicos brasileiros provavelmente produziria resultados mais precisos.
