# Chroma - Banco de Dados

**Função:** 

- Armazenar os dipositivos necessários para a execução do RAG (Retrieval-Augmented Generation).

**Visão Geral:** 

- Utiliza o Vanna.AI com ChromaDB como banco vetorial persistente para treinar um agente capaz de gerar queries SQL a partir de perguntas em linguagem natural.

## Fluxo Principal:

1. Configura e inicializa o ChromaDB persistente em disco.

2. Conecta ao Oracle Database para leitura de schemas (DDL).

3. Treina o modelo Vanna com:

    ✅ Estrutura do banco (DDL).

    ✅ Documentação - intruções para o modelo gerar SQLs.

    ✅ Exemplos de Perguntas e SQLs (JSON).

## Estrutura:
```
Projeto
├── Outras pastas do Projeto
├── chroma/
│   ├── chroma_db/
│   │       ├── repositório 1
│   │       ├── repositório 2
│   │       ├── repositório 3
│   │       └── chroma.sqlite3   (gerados automaticamente após rodar o train_vectorstore)
│   ├── ddl_vanna_oracle.txt
│   ├── documentation_bd_atual.txt
│   ├── README_CHROMA.md
│   ├── sql_oracle.json
│   ├── train_vectorstore.py
│   ├── verificar_colecoes_chroma.py
│   └── verificar_dados_chroma.py  

```
## chroma_db

**Função:** 

- Armazenar os vetores de embeddings em pastas e os metadados em um banco.

**Divisão:**  

- Dividido em 3 repositórios (coleções) referentes ao vetores de embeddings dos docuemntos - ddl_vanna_oracle.txt, documentation_bd_atual.txt e sql_oracle.json, e um banco de metadados - chroma.sqlite3.    

**chroma.sqlite3:** 

- É o banco de metadados que armazena as informações estruturais como - nomes das coleções
configuração da coleção (dimensionalidade, função de distância etc.), metadados dos documentos (IDs dos embeddings). Sua função é o mapeamento.

**repositórios (coleções):** 

- armazenam os vetores (embeddings), sendo cada coleção um banco vetorial isolado.

## ddl_vanna_oracle.txt

**Função:** 

- Definir a estrutura do banco de dados.

## documentation_bd_atual.txt

**Função:** 

- Atuar como propmt para o vanna na geração do SQL.

## sql_oracle.json

**Função:** 

- Atuar com exemplos de pares Perguntas e SQLs ideais para a geração do SQL.

**PONTO IMPORTANTE:** 

- O arquivo JSON deve ter esse formato:
```json
[
  { "question": "xxxxxx", "sql": "xxxxxx" },
]
```
## train_vectorstore.py

**Função:** 
    
- Conecta ao Oracle para autenticação.

- Treina o modelo Vanna com:

    1. Estrutura do banco (ddl_vanna_oracle.txt).

    2. Documentação (documentation_bd_atual.txt).

    3. Exemplos de Perguntas e SQLs (sql_oracle.json).

- Persiste os embeddings no ChromaDB para uso do banco localmente.

**Código:**

✅ Cria um cliente Chroma persistente.

✅ Carrega DDLs e treina o Vanna com elas.

✅ Carrega documentação textual e treina.

✅ Carrega exemplos de perguntas e SQL e treina.

✅ Armazena tudo em ChromaDB para uso futuro.

**Detalhamento do Código:**

**1. Importações:**
```bash
import oracledb
from vanna.openai import OpenAI_Chat
from vanna.chromadb.chromadb_vector import ChromaDB_VectorStore
import os
from dotenv import load_dotenv
import json
from chromadb import PersistentClient
```
- oracledb — driver para conexão com Oracle Database.

- OpenAI_Chat — backend do Vanna com OpenAI para geração de SQL.

- ChromaDB_VectorStore — backend do Vanna para armazenamento vetorial.

- os — manipulação de caminhos e variáveis de ambiente.

- dotenv — permite carregar variáveis do .env.

- json — usado para carregar dataset de perguntas + SQL.

- PersistentClient — cliente ChromaDB que salva os vetores em disco.

**2. Classe MyVanna:**
```bash
class MyVanna(ChromaDB_VectorStore, OpenAI_Chat):
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        OpenAI_Chat.__init__(self, config=config)
```
**Função:** 

- Cria uma classe que combina: ChromaDB como vetor store e OpenAI como gerador de SQL.

- Permite ao Vanna:

    1. Armazenar embeddings no Chroma

    2. Aprender com DDL, documentação e recuperar exemplos

    3. Gerar SQL com OpenAI

**3. Banco persistente do Chroma:**
```bash
chroma_client = PersistentClient(path=CHROMA_PATH)
```
**Função:** 

- Cria um repositório vetorial salvo em disco (localmente). 

- Garante que o modelo não precisa ser re-treinado a cada execução.

**4. Lista das coleções existentes no banco do Chroma:**
```bash
for col in chroma_client.list_collections():
    print("-", col.name)
```
**Função:** ver se o treinamento anterior foi salvo e verificar os nomes das coleções existentes.

**5. Vanna com OpenAI + Chroma:**
```bash
vn = MyVanna(config={
    'client': chroma_client,
    'api_key': os.getenv("OPENAI_API_KEY"),
    'model': "gpt-4.1",
})
```
- Instancia o objeto do Vanna com o modelo da OpenAI e o banco de dados do Chroma.

- Configura o Vanna para usar o modelo ali selecionado e usar o ChromaDB como armazenamento vetorial.

**6. Carregar DDL e treinar o Vanna com DDL:**

**Carrega o DDL:**
```bash
ddl_save_path = os.path.join(os.path.dirname(__file__), "ddl_vanna_oracle.txt")
with open(ddl_save_path, "r", encoding="utf-8") as f_save:
    ddls = f_save.read().strip().split("\n\n")
```
**Função:** 

- Lê um arquivo .txt contendo vários DDLs.

- Cada DDL é separado por uma linha em branco (\n\n).

- Retorna uma lista: [ddl1, ddl2, ddl3, ...]

**Treina o Vanna com DDL:**
```bash
for ddl in ddls:
    if ddl and ddl.strip():
        try:
            response = vn.train(ddl=ddl)
```
- O método 'vn.train()' faz com que o Vanna entenda a estrutura do banco (tabelas, colunas, etc.), gere os embeddings dessas estruturas esalve no ChromaDB.

**7. Carregar documentação e treinar o Vanna com a documentação:**

**Carrega a documentação:**
```bash
def load_documentation(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        return file.read()

file_path_documentation = os.path.join(os.path.dirname(__file__), "documentation_bd_atual.txt")

documentation_text = load_documentation(file_path_documentation)
```
- Recebe o caminho com arquivo da documentação em .txt

- Lê o arquivo 

- Retorna uma string contendo o texto completo da documentação.

**Treina o Vanna com a documentação:**
```bash
vn.train(documentation=documentation_text)
```
- O método 'vn.train()' recebe o parâmetro 'documentation' e então:

    1. Gera embeddings do texto da documentação

    2. Armazena esses embeddings no ChromaDB

    3. Conecta ao modelo OpenAI

- Quando o usuário fizer uma pergunta o Vanna consulta esses embeddings no processo de RAG para gerar o SQL.

**8. Carregar e Treinar Vanna com par de Perguntas + SQLs**

**Carrega o arquivo JSON:**
```bash
with open(os.path.join(os.path.dirname(__file__), 'sql_oracle.json'), 'r', encoding='utf-8') as file:
    questions_and_sql = json.load(file)
```

-  Lembrando que o arquivo JSON deve ter esse formato:
```json
[
  { "question": "xxxxxx", "sql": "xxxxxx" },
]
```

**Treina o Vanna com os exemplos de Pergunta + SQLs:**
```bash
for item in questions_and_sql:
    vn.train(question=item['question'], sql=item['sql'])
```
- Cria embeddings semânticos das Perguntas e dos SQLs que serão armazenados em uma coleção.

- Aumenta precisão da geração de SQLs.

## Resumo do train_vectorstore.py:

1. Carrega variáveis de ambiente

2. Cria um cliente Chroma Localmente

3. Cria uma classe Vanna usando OpenAI + Chroma

4. Lê DDLs e treina o modelo

5. Lê documentação e treina o modelo

6. Lê exemplos de perguntas + SQL e treina o modelo

7. Salva tudo em chroma_db nas suas respectivas coleções para usar no processo de RAG

## Como rodar o script train_vectorstore.py:

**Dependências:**

1. Crie ou acesse seu ambeinte .venv

2. Instale as dependências do projeto: 

```bash
python -m pip install -r requirements.txt
```

3. Caso precise - as dependências exclusivas para o train_vectorstore.py são:

```bash
pip install oracledb python-dotenv vanna chromadb openai
```

**Execução:**

1. Certifique-se de que os arquivos abaixo existam:

    ✅ ddl_vanna_oracle.txt → contém os DDLs do banco.

    ✅ documentation_bd_atual.txt → documentação extra para o treinamento.

    ✅ sql_oracle.json → Perguntas e SQLs para o RAG (formato Oracle).

2. Execute o script Python:

```bash
python python chroma/train_vectorstore.py
``` 

3. O banco vetorial será criado na pasta chroma/chroma_db/.

## Acrescentando novos dados:

- Para adicionar novos dados como Perguntas e SQLs ao banco já existente basta colocar em arquivo json as novas perguntas e seu respectivos sqls, seguindo o formato que está no arquivo sql_oracle.json. 

- As Perguntas e SQLs novos NÃO devem ser colocados nos mesmo arquivo que já foi subido para o banco local, ou seja, NÃO se deve misturar as Perguntas e SQLs já presentes no banco com as que você deseja adicionar, pois isso vai levar a uma duplicação das Perguntas e SQLs que já estão presentes no banco local.

- Assim, o ideal é criar um novo arquivo json com essas respectivas Perguntas e SQLs novos e passar seu caminho.

- Já o DDL e o Documantation não é necessário retirar, eles não serão duplicados. Se quiser passar um novo DDL ou um Novo Documantation basta alterar o arquivo que já possui e rodar normalmente, lembrnado que se for fazer apenas a mudança deles, deve-se levar em contar o que tem dentro do arquivo json para não duplicar Perguntas e SQLs.

- O Banco Local é dividido em 3 coleções:

    - ddl (retirado do arquivo - ddl_vanna_oracle.txt)
    - sql (retirado do arquivo - sql_oracle.json)
    - documentation (retirado do arquivo - documentation_bd_atual.txt)

- São as pastas que ficam juntas ao chroma.sqlite3 na pasta 'chroma_db'(não estão exatamente nessa ordem na pasta).

- Após considerar esses passos basta rodar novamente:

```bash
python python chroma/train_vectorstore.py
``` 

## Verificando novos dados e duplicatas no banco

- Para fazer a verificação se houve duplicatas no banco em alguma das coleções basta colocar o caminho do seu CHROMA_PATH e rodar no terminal:

```bash
python python chroma\verificar_colecoes_chroma.py
``` 
- Você terá um saída indicando a quantidade de docuemntos em cada coleção, serve para avaliar se foi ou não duplicado o ddl ou documantation.

- Para verificar especificamente a adição de novas Perguntas e SQLs na coleção basta colocar o caminho do seu CHROMA_PATH e rodar no terminal:

```bash
python python chroma\verificar_dados_chroma.py
``` 

## Detalhando os códigos de verificação:

### verificar_colecoes_chroma.py:

- Este script é usado para inspecionar o banco de vetores nas pastas em chroma_db e verificar quantos documentos existem em cada coleção usada pelo Vanna (ddl, sql, documentation).

**Função:** 

- Verificação da quantidade de coleções criadas no treinamento.

**1. Importações:**
```bash
import chromadb
import os
from dotenv import load_dotenv
```
- chromadb: Biblioteca do Chroma Vector Database (utilizada para armazenar embeddings).

- os: Usado para manipular caminhos de arquivos no sistema operacional.

- dotenv: Permite carregar variáveis do arquivo .env.

**2. Definir o caminho onde o 'chroma_db' está salvo:**
```bash
CHROMA_PATH = os.path.join(os.path.dirname(__file__), "chroma_db")
```

**3. Lista de coleções a serem inspecionadas:**
```bash
colecoes = ["ddl", "sql", "documentation"]
```
- Essas coleções correspondem ao treinamento feito anteriormente com Vanna.

**4. Loop para inspecionar cada coleção:**
```bash
for nome_colecao in colecoes:
    collection = client.get_or_create_collection(name=nome_colecao)
    results = collection.get(include=["documents"])
    print(f"{nome_colecao}: {len(results['documents'])} documentos")
```
- Se a coleção existe → apenas a abre.

- Se não existe → cria uma nova vazia.

- Recupera todos os documentos armazenados na coleção.

- Obtém o número de documentos~

- Imprime o total de documentos por coleção. Exemplo de saída:
```bash
Quantidade de documentos em cada coleção:

ddl: x documentos
sql: x documentos
documentation: x documentos
```

### verificar_dados_chroma.py:

- Este script é usado para inspecionar e visualizar os dados que estão dentro do banco de vetores nas pastas em chroma_db (ddl, sql, documentation).

**Função:** 

- Verificação dos dados do treinamento, avaliar se há duplicatas nos pares Perguntas e SQLs.

**1. Importações:**
```bash
import chromadb
import json
import os
from dotenv import load_dotenv
```
- chromadb: Biblioteca do Chroma Vector Database (utilizada para armazenar embeddings).

- json: Usado para converter strings para objeto Python.

- os: Usado para manipular caminhos de arquivos no sistema operacional.

- dotenv: Permite carregar variáveis do arquivo .env.

**2. Definir o caminho onde o 'chroma_db' está salvo e qual coleção será verificada:**
```bash
CHROMA_PATH = os.path.join(os.path.dirname(__file__), "chroma_db")

COLLECTION_NAME = "sql"
```

**3. Abrir/obter a coleção selecionada:**
```bash
collection = client.get_or_create_collection(name=COLLECTION_NAME)
```
- Se já existir, carrega.

- Se não existir, cria.

**4. Busca todos os itens da coleção:**
```bash
results = collection.get(include=["documents"])
```
- Retorna um dicionário com lista de textos (strings).

- Exemplo para os pares Pergunta e SQL:
```json
{
  "question": "xxxxxxx",
  "sql": "xxxxxxx"
}
```

- Essa parte faz um loop que exibe todos os dados dentro da coleção, ou seja, irá mostrar todos os pares de Perguntas e SQLs dentro da coleção 'sql':
```bash
for i, doc_str in enumerate(results["documents"]):
```

- A saída final será vista assim:
```
--- Item X ---
Pergunta: xxxxxxx
SQL: xxxxxxxx

```
