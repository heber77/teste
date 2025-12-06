# Features - Nomes e abreviações das entidades

- Contém nomes e abreviações das entidades (CULTIVAR, GENETICA, TECNOLOGIA, DESCRICAO_CIDADE, ESTADO, RESISTENCIAS e SAFRA), o código que faz a correção dessas entidades e os parâmetros de similaridade para que isso ocorra.

- Dentro do json feature_entidade.json estão os nomes das entidades coforme o disposto corretamente no Banco de dados real.

- Dentro do json feature_map_entidade.json estão as abrviações, que não tenham nomes iguais/duplicados, equivalentes aos nomes corretos do Banco de dados real.

- Dentro do json feature_map_entidade_duplicada.json estão os nomes iguais/duplicados equivalentes aos nomes corretos do Banco de dados real.

**Exemplificando as abrviações e duplicatas de nomes:**

**Abreviações:**
```json
"GUEPARDO": "BRASMAX GUEPARDO IPRO",
```

**Duplicatas de nomes:**
```json
"IGUACU": ["HO IGUACU IPRO", "FPS IGUACU RR"],
```

**Exemplificando os parâmetros:**
```json
"CULTIVAR": {
    "threshold": 85
  },
```
- Os parêmetros/threshold servem para barrar nomes que não atinjam a similaridade necessária para a correção, ou seja, ele delimita quais nomes podem ou não ser similares ao nome que é realmente correto no banco de dados.

## Corretor de Features - FeatureMatcher (normalizer_feature_matcher.py):

**Função:** 

- Realizar a correção das entidades por meio do algorítmo Fuzzy.

**1. Importações:**
```bash
import re
import json
import sqlite3
import os
from thefuzz import process, fuzz
from unidecode import unidecode
import oracledb
from dotenv import load_dotenv
```
- re: Usada para extrair padrões de texto, normalizar strings, validar formatos específicos, remover caracteres indesejados.

- json: utilizada para, ler arquivos '.json', converter objetos Python para JSON, interpretar strings JSON vindas de bancos.

- sqlite3: Usada para abrir conexões com '.db' locais, executar consultas SQL, integrar SQLite ao pipeline (ex.: pré-validação, cache, índices, RAG local).

- os: Usada para criar caminhos de arquivos 'os.path.join', ler variáveis de ambiente, descobrir diretório atual (__file__), verificar existência de arquivos e pastas.

- thefuzz/process, fuzz: Importa o fuzzy matching, usado para comparação de similaridade de strings. Utilizada para comparação aproximada entre texto do usuário e nomes de colunas/tabelas, encontrar o item mais parecido com base em similaridade, criar mecanismos de tolerância a erro no processamento de texto.

- unidecode: Usada para padronizar textos para comparação (ex.: "São Paulo" → "Sao Paulo"), melhorar precisão do fuzzy matching, normalizar entradas vindas de SQL ou mensagens do usuário.

- oracledb: Utilizado para criar conexões com bancos Oracle, executar queries diretamente, ler DDLs, metadados e estruturas de tabelas, integrar Oracle ao pipeline RAG ou geradores de SQL.

- dotenv/load_dotenv: Usada para carregar credenciais (senhas, usuário, DSN), configurar chaves de API de forma segura, manter o código limpo e sem informação sensível.

**2. Class FeatureMatcher:**

**Função:**

- A classe FeatureMatcher é responsável por corrigir e padronizar as entidades extraídas do SQL.

**Estrutura Geral:**

1. Carregar parâmetros e dados dos arquivos JSON.

2. Normalizar texto (tirar acentos, caracteres especiais, caixa).

3. Corrigir abreviações com base nos arquivos JSON pré-definidos.

4. Aplicar fuzzy matching para encontrar a entidade mais similar, considerando os parêmetros/threshold.

5. Consultar o Oracle para validar features (FAZENDA e TALHAO) dinamicamente com os dados no banco Oracle do cliente.

6. Corrigir o SQL completo, identificando entidades com erro e substituindo por valores válidos.

**3. Carregando variáveis de ambiente (.env), dados e parâmetros dos JSONs:**
```bash
def __init__(self):
    load_dotenv()
    self.parameters = self._load_parameters()
    self.feature_maps = self._load_feature_maps()
```
- Carrega variáveis de ambiente do arquivo .env usando load_dotenv().

- Inicializa self.parameters carregando automaticamente os valores do arquivo parameters.json.

- Inicializa self.feature_maps carregando todos os arquivos feature_map_*.json encontrados na pasta features.

```bash
def _load_parameters(self) -> dict:
    params_path = os.path.join(BASE, 'parameters.json')
    with open(params_path, 'r', encoding='utf-8') as f:
        return json.load(f)
```
- Monta o caminho para o arquivo parameters.json.

- Abre o arquivo e lê seu conteúdo.

- Retorna um dicionário contendo parâmetros usados (Thresholds para fuzzy matching).

```bash
def _load_feature_data(self, feature_type: str):
    file_path = os.path.join(BASE, f'feature_{feature_type.lower()}.json')
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            feature_data = json.load(f)
        return feature_data.get(feature_type.upper(), [])
    except FileNotFoundError:
        return None
```
- Recebe um nome de feature, por exemplo "cidade".

- Constrói um caminho como: feature_cultivar.json

- Tenta abrir e ler o arquivo.

- Dentro do JSON, busca a chave em letras maiúsculas: "CULTIVAR": [...]

- Retorna a lista de valores/itens relacionados a essa feature.

- Faz isso para cada feature.json.

```bash
def _load_feature_maps(self) -> dict:
    feature_maps = {}
    for fname in os.listdir(BASE):
        if fname.startswith("feature_map_") and fname.endswith(".json"):
            feature_key = fname[len("feature_map_"):-len(".json")]
            with open(os.path.join(BASE, fname), encoding="utf-8") as f:
                feature_maps[feature_key] = json.load(f)
    return feature_maps
```
- Carrega todos os arquivos JSON de mapeamento de features (feature_map_entidade.json)

- Percorre todos os arquivos na pasta.

- Seleciona arquivos com padrão: feature_map_entidade.json

- Extrai a entidade e usa como chave do dicionário.

- Abre o arquivo e carrega seu conteúdo JSON.

- Armazena tudo em: self.feature_maps.

**4. Limpeza e matching:**
```bash
def _clean_text(self, text) -> str:
    if not isinstance(text, str):
        text = str(text)
    text = unidecode(text)
    cleaned = re.sub(r'[^a-zA-Z0-9 ]', '', text)
    return cleaned.lower()
```
- Normaliza o texto para facilitar comparações, fuzzy matching e padronizar o formato da escrita para evitar erros na correção.

- Garante que a entrada é string. Caso não seja (int, None, etc.), converte para string.

- Remove acentuação com o unidecode, transforma: "São Paulo" → "Sao Paulo".

- Remove caracteres especiais via regex: [^a-zA-Z0-9 ]. São removidos - pontuação, símbolos, quebras de linha, barras, parênteses etc.

    - Exemplo: "A/B - 45!" → "AB 45".

- Converte para lowercase. Assim, deixa todas as comparações case-insensitive (mais segura para evitar erros).

- Resultado final: Um texto limpo, sem acentos, sem símbolos e sempre em minúsculas.

    - Exemplo:

        - Entrada: "Café-Do-João 2024!"

        - Saída: "cafedojoao 2024"

```bash
def _find_closest_match(self, cleaned_input: str, cleaned_known: list, raw_entities: list, similarity_threshold: int, limit: int = 5):
    similar = process.extract(cleaned_input, cleaned_known, limit=limit, scorer=fuzz.token_sort_ratio)
    for cl, score in similar:
        if score >= similarity_threshold:
            raw = next(r for r, c in zip(raw_entities, cleaned_known) if c == cl)
            return raw, None

    suggestions = []
    for cl, score in similar:
        raw = next(r for r, c in zip(raw_entities, cleaned_known) if c == cl)
        suggestions.append((raw, score))
    return None, suggestions
```
- Realiza um fuzzy matching (busca de similaridade) do texto normalizado do usuário comparando-o com as listas de valores conhecidos retirados dos 'arquivos.json'.

- Pode retornar:

    1. Um match perfeito o suficiente - quando encontra um nome que bate o valor mínimo de threshold. (Pega o mais similar)

    2. Ou sugestões, caso nenhum item passe o threshold mínimo. (Retorna até 5 sugestões, menos para SAFRA que seram apenas 3)


**5. Correção de abreviações e nomes das entidades:**
```bash
def _correct_abbreviation(self, feature_type: str, value: str) -> str:
        threshold = self.parameters.get(feature_type)['threshold']
        feature_type_lower = feature_type.lower()
        if feature_type_lower not in self.feature_maps:
            return value

        feature_map = self.feature_maps[feature_type_lower]
        # === CULTIVAR ===
        if feature_type.upper() == "CULTIVAR":
            if "ABREVIACOES_CULTIVARES_UNICAS" in feature_map:
                mapping = feature_map["ABREVIACOES_CULTIVARES_UNICAS"]
                if value in mapping:
                    return mapping[value]
                else:
                    best = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return mapping[best[0]]
            if "ABREVIACOES_CULTIVARES_DUPLICADAS" in feature_map:
                mapping = feature_map["ABREVIACOES_CULTIVARES_DUPLICADAS"]
                if value in mapping:
                    best = process.extractOne(value, mapping[value], scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return best[0]
                else:
                    best_key = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best_key and best_key[1] >= threshold:
                        best = process.extractOne(value, mapping[best_key[0]], scorer=fuzz.token_sort_ratio)
                        if best and best[1] >= threshold:
                            return best[0]
        # === CIDADE ===
        elif feature_type.upper() == "DESCRICAO_CIDADE":
            if "ABREVIACOES_CIDADES_UNICAS" in feature_map:
                mapping = feature_map["ABREVIACOES_CIDADES_UNICAS"]
                if value in mapping:
                    return mapping[value]
                else:
                    best = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return mapping[best[0]]
            if "ABREVIACOES_CIDADES_DUPLICADAS" in feature_map:
                mapping = feature_map["ABREVIACOES_CIDADES_DUPLICADAS"]
                if value in mapping:
                    best = process.extractOne(value, mapping[value], scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return best[0]
                else:
                    best_key = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best_key and best_key[1] >= threshold:
                        best = process.extractOne(value, mapping[best_key[0]], scorer=fuzz.token_sort_ratio)
                        if best and best[1] >= threshold:
                            return best[0]
        # === GENÉTICA / RESISTÊNCIAS / TECNOLOGIA ===
        elif feature_type.upper() == "GENETICA":
            if "ABREVIACOES_GENETICA" in feature_map:
                mapping = feature_map["ABREVIACOES_GENETICA"]
                if value in mapping:
                    return mapping[value]
                else:
                    best = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return mapping[best[0]]
        elif feature_type.upper() == "RESISTENCIAS":
            if "ABREVIACOES_RESISTENCIAS" in feature_map:
                mapping = feature_map["ABREVIACOES_RESISTENCIAS"]
                if value in mapping:
                    return mapping[value]
                else:
                    best = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return mapping[best[0]]
        elif feature_type.upper() == "TECNOLOGIA":
            if "ABREVIACOES_TECNOLOGIA" in feature_map:
                mapping = feature_map["ABREVIACOES_TECNOLOGIA"]
                if value in mapping:
                    return mapping[value]
                else:
                    best = process.extractOne(value, list(mapping.keys()), scorer=fuzz.token_sort_ratio)
                    if best and best[1] >= threshold:
                        return mapping[best[0]]
        return value
```
**Função:**

- Corrigir abreviações, grafias incompletas ou variações de nomes de acordo com os maps JSON específicos por tipo de entidade.

- Usa: Mapeamentos diretos (quando existe uma substituição exata), Fuzzy-matching (quando a escrita é aproximada), Categorias especiais (duplicadas, únicas etc.).

**Estrutura geral:**

1. Carrega o threshold de similaridade:
```bash
threshold = self.parameters.get(feature_type)['threshold']
```
    - Cada feature tem sua própria tolerância, definida no parameters.json.

2. Verifica se existe map para cada 'feature_type':
```bash
feature_type_lower = feature_type.lower()
        if feature_type_lower not in self.feature_maps:
            return value
```
    - Se não houver mapeamento, o valor original é retornado sem alterações.

3. Obtém o map de abreviações:
```bash
feature_map = self.feature_maps[feature_type_lower]
```
    - Cada feature tem seu arquivo 'feature_map_entidade.json'.

4. Aplica lógica de correção específica para cada tipo de feature:

- CULTIVAR: Possui map para abreviações únicas (sem nomes iguais) e duplicadas.

    - Exemplo:
    ```json
    "ABREVIACOES_CULTIVARES_UNICAS": {
        "GUEPARDO": "BRASMAX GUEPARDO IPRO",
        "60177": "LG 60177 IPRO",
    }
    ```

    ```json
    "ABREVIACOES_CULTIVARES_DUPLICADAS": {
        "8383": ["GH 8383 RR", "BRS 8383 IPRO", "NS 8383 RR"],
    }
    ```
    - Consideramos duplicadas aquele valores que possuam mais de um valor igual.

    **Fluxo:**

    1. Se o valor existe como chave → tenta fuzzy dentro das opções.

    2. Se não existe → fuzzy para determinar qual chave é mais próxima → depois fuzzy dentro da lista de cultivares.

- DESCRICAO_CIDADE: Possui map para abreviações únicas (sem nomes iguais) e duplicadas.

    - Exemplo:
    ```json
    "ABREVIACOES_CIDADES_UNICAS": {
        "Santa Helena": "Santa Helena de Goias",
        "Marianopolis": "Marianopolis do Tocantins",
    }
    ```

    ```json
    "ABREVIACOES_CIDADES_DUPLICADAS": {
    "Rio Verde": ["Lucas do Rio Verde", "Rio Verde", "Rio Verde de Mato Grosso"],
    }
    ```
    **Fluxo:**

    1. Se o valor existe como chave → tenta fuzzy dentro das opções.

    2. Se não existe → fuzzy para determinar qual chave é mais próxima → depois fuzzy dentro da lista de cidades.

- GENETICA, RESISTENCIAS, TECNOLOGIA: Possuem apenas um dicionário, sem valores de nomes duplicados.

    - Usam fuzzy básico + threshold.

    - Exemplo (GENETICA):
    ```json
    "ABREVIACOES_GENETICA": {
        "SYN": "Syngenta",
        "DM": "DonMario",
    }
    ```

    - Exemplo (RESISTENCIAS):
    ```json
    "ABREVIACOES_RESISTENCIAS": {
        "Olho-de-rã": "DOENÇA Mancha-olho-de-rã",
        "Podridão": "DOENÇA Podridão Radicular de Fitóftora",
    }
    ```

    - Exemplo (TECNOLOGIA):
    ```json
    "ABREVIACOES_TECNOLOGIA": {
        "XTD":"Xtend",
        "I2X":"Intacta 2 Xtend",
    }
    ```

    **Fluxo:**

    1. Se existe mapeamento direto → retorna.

    2. Senão → fuzzy entre as chaves.

    3. Se score ≥ threshold → retorna o valor mapeado.

**6. Correção genérica de features:**
```bash
def correct_feature_list(self, feature_type: str, features: list) -> tuple:
        feature_type = feature_type.upper()
        if feature_type not in self.parameters:
            return features, None

        known_entities = self._load_feature_data(feature_type)
        if not known_entities:
            return features, None

        similarity_threshold = self.parameters.get(feature_type)['threshold']
        cleaned_entities = [(ent, self._clean_text(ent)) for ent in known_entities]
        cleaned_known = [cl for _, cl in cleaned_entities]
        corrected_list = []

        entity_friendly = {
            "CULTIVAR": "cultivar",
            "TECNOLOGIA": "tecnologia",
            "GENETICA": "genética",
            "DESCRICAO_CIDADE": "cidade",
            "ESTADO": "estado",
            "RESISTENCIAS": "resistência",
            "SAFRA": "safra"
        }
        friendly = entity_friendly.get(feature_type, feature_type.lower())

        for value in features:
            if feature_type == 'SAFRA':
                match_full = re.match(r'^\d{4}/\d{4}$', value)
                match_year = re.match(r'^\d{4}$', value)
                if match_full:
                    corrected_list.append(value)
                elif match_year:
                    corrected = f"{value}/{int(value)+1}"
                    corrected_list.append(corrected)
                else:
                    value_corrected = self._correct_abbreviation(feature_type, value)
                    cleaned_value = self._clean_text(value_corrected)
                    limit_sug = 3
                    corrected_name, suggestions = self._find_closest_match(cleaned_value, cleaned_known, known_entities, similarity_threshold, limit_sug)
                    if corrected_name is None:
                        top_suggestions = [f"{raw}" for raw, score in suggestions]
                        msg = (
                            f"Desculpe, mas não encontrei {friendly} com o nome '{value}'. "
                            f"Será que você quis dizer: {', '.join(top_suggestions)}?"
                        )
                        return [], msg
                    corrected_list.append(corrected_name)
            else:
                value = self._correct_abbreviation(feature_type, value)
                cleaned_value = self._clean_text(value)
                limit_sug = 5
                corrected_name, suggestions = self._find_closest_match(cleaned_value, cleaned_known, known_entities, similarity_threshold, limit_sug)
                if corrected_name is None:
                    top_suggestions = [f"{raw}" for raw, score in suggestions]
                    msg = (
                        f"Desculpe, mas não encontrei {friendly} com o nome '{value}'. "
                        f"Será que você quis dizer: {', '.join(top_suggestions)}?"
                    )
                    return [], msg
                corrected_list.append(corrected_name)
        return corrected_list, None
```
**Estrutura:**

1. Normaliza o tipo de feature. 

2. Carrega os valores conhecidos.

    - Exemplo: para CULTIVAR, carrega a lista de cultivares existentes no JSON.

3. Obtém o limiar de similaridade (threshold).

4. Pré-processa entidades conhecidas, facilitando a comparação com as entradas do usuário.

5. Mapeamento de nomes para montar mensagens amigáveis ao usuário em casos de não encontrar a feature.
 ```json
entity_friendly = {
    "CULTIVAR": "cultivar",
    "TECNOLOGIA": "tecnologia",
    "GENETICA": "genética",
    "DESCRICAO_CIDADE": "cidade",
    "ESTADO": "estado",
    "RESISTENCIAS": "resistência",
    "SAFRA": "safra"
}
```
    - Exemplo: “Não encontrei cidade com o nome...”

6. Laço principal: corrige cada valor recebido
```bash
for value in features:
    if feature_type == 'SAFRA':
        match_full = re.match(r'^\d{4}/\d{4}$', value)
        match_year = re.match(r'^\d{4}$', value)
        if match_full:
            corrected_list.append(value)
        elif match_year:
            corrected = f"{value}/{int(value)+1}"
            corrected_list.append(corrected)
        else:
            value_corrected = self._correct_abbreviation(feature_type, value)
            cleaned_value = self._clean_text(value_corrected)
            limit_sug = 3
            corrected_name, suggestions = self._find_closest_match(cleaned_value, cleaned_known, known_entities, similarity_threshold, limit_sug)
            if corrected_name is None:
                top_suggestions = [f"{raw}" for raw, score in suggestions]
                msg = (
                    f"Desculpe, mas não encontrei {friendly} com o nome '{value}'. "
                    f"Será que você quis dizer: {', '.join(top_suggestions)}?"
                )
                return [], msg
            corrected_list.append(corrected_name)
    else:
        value = self._correct_abbreviation(feature_type, value)
        cleaned_value = self._clean_text(value)
        limit_sug = 5
        corrected_name, suggestions = self._find_closest_match(cleaned_value, cleaned_known, known_entities, similarity_threshold, limit_sug)
        if corrected_name is None:
            top_suggestions = [f"{raw}" for raw, score in suggestions]
            msg = (
                f"Desculpe, mas não encontrei {friendly} com o nome '{value}'. "
                f"Será que você quis dizer: {', '.join(top_suggestions)}?"
            )
            return [], msg
        corrected_list.append(corrected_name)
return corrected_list, None
```
    - Caso especial: SAFRA

        - Safra tem regras próprias porque pode ser:

            1. '2023/2024' (válido).

            2. '2023' (corrigir para '2023/2024').

            3. Nome abreviado como '23/24' (tenta corrigir via função auxiliar).

        - Se já está no formato completo aceita direto.

    - Demais features (CULTIVAR, DESCRICAO_CIDADE, GENETICA, etc.)

        1. Corrige abreviações.

        2. Limpa o texto.

        3. Busca match via similaridade.

        4. Se não encontrar → sugere as mais próximas.

            - Exemplo: 
                
                "Desculpe, mas não encontrei cidade com o nome... . "
                "Será que você quis dizer: ...?"

    - Por fim retorna valores corrigidos.

7. Consulta aos dados no banco Oracle
```bash
def _query_oracle(self, feature_type: str, client_id: str, cultura_id: str = None) -> list:
        try:
            user = os.getenv('USER')
            password = os.getenv('SENHA')
            dsn = os.getenv('DSN')
            conn = oracledb.connect(user=user, password=password, dsn=dsn)
            cursor = conn.cursor()
            column = feature_type.upper()
            query = f"SELECT DISTINCT {column} FROM plantup.VW_PLANTUP WHERE CLIENTE_ID = :cliente_id"
            params = {'cliente_id': client_id}
            if cultura_id is not None:
                query += " AND CULTURA_ID = :cultura_id"
                params['cultura_id'] = cultura_id
            query += f" ORDER BY {column}"
            cursor.execute(query, params)
            raw_results = [row[0] for row in cursor.fetchall() if row[0] is not None]
            cleaned_results = [(str(orig), self._clean_text(str(orig))) for orig in raw_results]
            cursor.close()
            conn.close()
            return cleaned_results
        except Exception as e:
            print(f"ERRO no Oracle para {feature_type}: {e}")
            return []
```
- Consulta o banco Oracle e retorna uma lista de valores de um determinado feature (coluna). 

- Usamos para fazer a correção de FAZENDA, TALHAO e sugerir SAFRA presente no banco do usuário.

- Filtrados por:

    1. CLIENTE_ID 

    2. CULTURA_ID

- Usamos as credenciais do Oracle para conectar via VPN:
```bash
user = os.getenv('USER')
password = os.getenv('SENHA')
dsn = os.getenv('DSN')

conn = oracledb.connect(user=user, password=password, dsn=dsn)
cursor = conn.cursor()
```
    - Cria a conexão e o cursor para executar queries.

- Usamos a query para consultar no banco:
```bash
query = f"SELECT DISTINCT {column} FROM plantup.VW_PLANTUP WHERE CLIENTE_ID = :cliente_id"
```

- Executa a query e extrai os resultados. 

- Limpa os resultados e retorna eles prontos.

- Em caso de erro, imprime a mensagem de erro e retorna vazio.

8. Correção do SQL (<def correct_sql>):

- A função <correct_sql> recebe uma SQL e corrige valores incorretos de colunas como:

    1. CULTIVAR

    2. TECNOLOGIA

    3. GENETICA

    4. DESCRICAO_CIDADE

    5. ESTADO

    6. RESISTENCIAS

    7. SAFRA

    8. FAZENDA (via Oracle)

    9. TALHAO (via Oracle)

- Ela faz isso seguindo um processo em várias etapas:

    **1. Detectar onde estão os valores a corrigir:**

    - A função escaneia a SQL procurando:

        **A. IN(...)**
        ```bash
            AND CULTIVAR IN ('soja', 'soja suprema')
        ```

        **B. Igualdades (=)**
        ```bash
            ESTADO = 'sp'
            SAFRA = '2020'
        ```

    - Ela separa cada entidade (coluna) e os valores associados.

    **2. Identificar dependências obrigatórias:**

    - Antes de corrigir:

        1. Descobre o CLIENTE_ID da query

        2. Descobre o CULTURA_ID

        3. Verifica se a SQL cita FAZENDA, TALHÃO, SAFRA

            - Se sim, consulta o Oracle para obter as listas válidas do cliente:
            ```bash
            known_fazendas = self._query_oracle('FAZENDA', client_id)
            known_talhoes  = self._query_oracle('TALHAO', client_id)
            known_safras   = self._query_oracle('SAFRA', client_id)
            ```
    **3. Lógica geral de correção de features:**

        1. Normalizar entrada:

            - Remover acentos

            - Tudo minúsculo

            - Só letras e números

        2. Encontrar match mais próximo:

            - Usado fuzzy matching (thefuzz) com token_sort_ratio.

            - Se a similaridade ≥ threshold definido nos parâmetros: → corrige automaticamente

            - Caso contrário: → retorna sugestões e não altera o SQL.

        3. Reescrever o SQL

            - Se um valor foi corrigido, a função faz:
            ```bash
            sql_query = sql_query.replace(original, corrected)
            ```
            - Então o valor é escrito corretamente no mesmo lugar dentro do SQL.

    **4. Correções específicas por tipo de feature:**

        1. SAFRA:

            - Formatos aceitos automaticamente: 
            
                1. 2020/2021 (mantém).

                2. 2020 → vira 2020/2021.

                3. Se o valor corrigido não existir no Oracle, tenta fuzzy matching.

                4. Se não achar → retorna sugestões com base nos dados do banco do usuário, ou caso tenha algum problema no banco usa os dados no JSON.

        2. FAZENDA e TALHÃO:

            - Dados consultados via Oracle.

            - Tenta o fuzzy matching.

            - Se não atingir threshold → retorna sugestões com base nos dados do usuário.

        3. RESISTÊNCIAS:

            - Tem uma regra especial pois a forma de detecção no SQL é diferente:

                - Detecta padrões como no SQL: 
                ```
                    r.'nome_resistencia' IN ('R', 'MR') ou r."nome_resistencia" IN ("R", "MR")
                ```
        
        4. Outras entidades (CULTIVAR, TECNOLOGIA, GENETICA etc.):

            - Seguem o fluxo genérico.

- A função retorna:

    1. sql_corrigida → string da query ajustada

    2. entidades_corrigidas → lista dos tipos alterados

    Exemplo: ["CULTIVAR", "SAFRA"]

- Resumo:

    - A função percorre a SQL inteiro identificando as colunas com valores das entidades, normaliza cada valor, tenta encontrar o match mais próximo usando fuzzy matching (seja pegando dos JSONs ou diretamente do banco Oracle do usuário - fazenda/talhão/safra), corrige automaticamente se a similaridade for maior ou igual ao limiar (threshold), e caso não encontre retorna sugestões para o usuário.

