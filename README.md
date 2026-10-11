# Case - Laceda

Pipeline local para relacionar endereços de uma base de consulta ao setor
censitário correspondente na base CNEFE da Bahia (IBGE).

O processamento é organizado em quatro notebooks e segue o fluxo:

```text
data/raw  ->  data/silver  ->  data/gold
  CSV          Parquet         CSV + Parquet
```

## Objetivo

Para cada registro da base de consulta, localizar o melhor endereço
correspondente no CNEFE e retornar o código do setor censitário, preservando
também os casos que exigem revisão manual.

O CNEFE possui múltiplas linhas para um mesmo endereço ou setor. Por isso, o
pipeline não depende de uma correspondência textual simples: ele normaliza os
campos, gera candidatos por *blocking*, calcula similaridades e aplica regras
de decisão conservadoras.

## Arquitetura do repositório

```text
.
├── README.md
├── requirements.txt
├── notebooks/
│   ├── 00_download_e_ambiente.ipynb
│   ├── 01_exploracao.ipynb
│   ├── 02_normalizacao.ipynb
│   └── 03_matching.ipynb
├── data/
│   ├── raw/       # CSVs de entrada; não versionados
│   ├── silver/    # Parquets normalizados; gerados pelo notebook 02
│   └── gold/      # resultados finais; gerados pelo notebook 03
└── 2026_teste_pratico_relinke_cidacs.pdf
```

Os arquivos de dados podem ser grandes e estão ignorados pelo Git. O PDF
contém o contexto do teste técnico e as referências para as bases de entrada.

## Tecnologias

- **Python 3.12** no ambiente utilizado para os notebooks.
- **Polars** para leitura lazy, transformação e gravação das bases
  intermediárias em Parquet.
- **DuckDB** para geração de candidatos, cálculo do ranking, auditoria e
  exportação dos resultados.
- **Jupyter Notebook** para execução reprodutível e inspeção das etapas.

Polars e DuckDB foram usados em vez de Spark ou Elasticsearch porque o volume
do CNEFE da Bahia é processado localmente de forma eficiente, com leitura lazy,
persistência em Parquet e geração de candidatos por *blocking*.

## Dados de entrada

Coloque em `data/raw/` os dois arquivos usados pelo pipeline:

```text
29_BA.csv
2026_teste_tecnico_relinke_cidacs(in).csv
```

O arquivo `29_BA.csv` é o CNEFE da Bahia, separado por `;`. A base de consulta
também é lida como CSV separado por `;` e deve conter as colunas:

```text
consulta_logradouro
consulta_numero
consulta_bairro
consulta_cep
consulta_complemento
consulta_municipio
```

O notebook `00_download_e_ambiente.ipynb` pode baixar os arquivos previstos
para `data/raw/` a partir dos identificadores configurados nele. Ele ignora
arquivos que já existam e grava primeiro em um arquivo temporário.

## Execução

### 1. Criar o ambiente e instalar dependências

No Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

### 2. Executar os notebooks na ordem

Execute as células na seguinte sequência:

1. [00_download_e_ambiente.ipynb](notebooks/00_download_e_ambiente.ipynb) —
   prepara o ambiente e baixa as bases, se necessário.
2. [01_exploracao.ipynb](notebooks/01_exploracao.ipynb) — verifica os esquemas
   e amostras das entradas.
3. [02_normalizacao.ipynb](notebooks/02_normalizacao.ipynb) — cria as bases
   normalizadas em `data/silver/`.
4. [03_matching.ipynb](notebooks/03_matching.ipynb) — calcula as
   correspondências e grava os resultados em `data/gold/`.

Para abrir os notebooks:

```powershell
jupyter notebook
```

Os caminhos relativos usados nos notebooks pressupõem a execução a partir da
pasta `notebooks/` ou um diretório de trabalho equivalente configurado pelo
Jupyter.

## Estratégia de normalização

As mesmas regras são aplicadas à base CNEFE e à base de consulta:

- leitura dos campos como texto para preservar códigos e valores especiais;
- conversão para maiúsculas;
- remoção de acentos, pontuação e espaços duplicados;
- conversão de números por extenso para dígitos;
- expansão de abreviações de tipos de logradouro, como `R`, `AV` e `TV`;
- limpeza de CEP, mantendo somente CEPs com oito dígitos;
- extração do número do endereço, tratando `SN`, `S/N` e `SEM NÚMERO`;
- normalização de bairro, localidade e complemento;
- remoção de classificações genéricas como `RURAL` quando elas não ajudam a
  diferenciar os candidatos.

Além dos campos normalizados, são criadas chaves auxiliares:

- `cep_limpo`;
- `cep5`, com os cinco primeiros dígitos do CEP;
- `logr_norm`, logradouro normalizado;
- `logr_core`, logradouro sem o tipo da via;
- `k_core`, prefixo usado no bloqueio por núcleo do logradouro.

## Estratégia de matching

O notebook `03_matching.ipynb` usa DuckDB para:

1. carregar as bases silver como views;
2. gerar candidatos no mesmo município por três blocos:
   - CEP exato;
   - `cep5`;
   - prefixo `k_core` do logradouro;
3. eliminar pares duplicados;
4. calcular similaridade Jaro-Winkler para logradouro, núcleo, bairro e
   complemento;
5. calcular pontuação de CEP e número;
6. combinar as pontuações com pesos renormalizados conforme os campos
   disponíveis;
7. manter os dois melhores candidatos por consulta;
8. aplicar regras de decisão conservadoras.

O score composto usa, quando disponíveis:

| Campo | Peso |
| --- | ---: |
| Logradouro | 0,42 |
| Núcleo do logradouro | 0,13 |
| Bairro/localidade | 0,15 |
| CEP | 0,05 |
| Número | 0,20 |
| Complemento | 0,05 |

Quando a consulta possui número, o resultado encontrado exige número exato.
Consultas sem número dependem de núcleo do logradouro e da diferença entre o
primeiro e o segundo candidato. Casos sem confiança suficiente são classificados
como ambíguos ou não encontrados, em vez de receberem uma atribuição forçada.

## Saídas

O notebook `02_normalizacao.ipynb` gera:

```text
data/silver/cnefe_ba.parquet
data/silver/busca.parquet
```

O notebook `03_matching.ipynb` gera:

```text
data/gold/resultado_final.csv
data/gold/resultado_final.parquet
```

O arquivo final contém as sete colunas de entrega:

```text
consulta_logradouro
consulta_numero
consulta_bairro
consulta_cep
consulta_complemento
consulta_municipio
setor_censitario_encontrado
```

Durante a execução, a tabela interna `resultado` também contém `status_match`,
`score_match` e os campos do melhor candidato. A auditoria do notebook exibe
os casos `encontrado`, `ambiguo` e `nao_encontrado`, mas esses campos de
auditoria não são exportados para o arquivo final atual.

## Métricas observadas na execução registrada

Na execução registrada nos notebooks:

| Métrica | Quantidade | Percentual |
| --- | ---: | ---: |
| Registros de entrada | 107 | 100,00% |
| `encontrado` | 28 | 26,17% |
| `ambiguo` | 79 | 73,83% |
| `nao_encontrado` | 0 | 0,00% |

Quantidade de pares candidatos por bloco antes do scoring:

| Estratégia | Pares |
| --- | ---: |
| CEP exato | 1.429.795 |
| `cep5` | 1.632.325 |
| Núcleo do logradouro | 118.416 |

Essas métricas são referentes à execução registrada nas saídas dos notebooks.
Para atualizar os números após uma nova execução, consulte a distribuição
impressa no final do notebook `03_matching.ipynb`.


## Limitações

- Não há coordenadas na base de consulta para desempate espacial.
- Endereços rurais, fazendas, rodovias e logradouros genéricos podem gerar
  muitos candidatos ou baixa similaridade.
- CEP, bairro e complemento podem estar ausentes ou divergentes entre as
  bases.
- Não há gabarito independente no repositório para estimar a acurácia real.
- O CNEFE completo da Bahia exige espaço em disco e memória suficientes para
  leitura e materialização dos candidatos.

## Reprodutibilidade

O ambiente Python e as dependências estão declarados em
[requirements.txt](requirements.txt). Os dados brutos, intermediários e
resultados são ignorados pelo Git conforme [.gitignore](.gitignore), portanto
uma nova execução deve baixar ou disponibilizar novamente os CSVs de entrada.
