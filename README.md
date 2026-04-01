# TrabalhoSO
O trabalho de SO
# handin_mgr

Script em `/bin/sh` para:
- Manter uma lista de tarefas persistente (to‑do list).
- Gerir entregas de trabalhos práticos (hand‑ins) organizadas por UC, TP e aluno.

O objetivo é simular uma ferramenta que um docente poderia usar para organizar e validar entregas dos alunos, ao mesmo tempo que treina o uso de shell POSIX e ferramentas standard de sistema.

---

## Autores

- Ryan Possan Rubi  
- Tiago Rodrigues  
- António Valente  

Licenciatura em Engenharia Informática Aplicada  
Ano letivo 2025/2026

---

## Requisitos

- Sistema Unix/Linux com:
  - `/bin/sh` (POSIX sh, sem funcionalidades específicas de `bash`).
  - Comandos standard: `find`, `grep`, `awk`, `sed`, `sort`, `uniq`, `cut`, `tr`, `wc`, `date`, `du`, `cp`, `mv`, `rm`, `mkdir`, `test`.

Estado é guardado na pasta do utilizador (`$HOME`), por isso o programa precisa de permissão de escrita em:

```text
$HOME/.handin_mgr/
```

Ferramentas opcionais:
- A versão atual **não usa** `tar`, `gzip` nem `unzip`.  
  Se existirem ficheiros comprimidos (`.zip`, `.tar`, `.tar.gz`), devem ser extraídos manualmente antes de correr o `handin_mgr`.

---

## Instalação e execução

1. Clonar o repositório:
   ```sh
   git clone https://github.com/toraval/TrabalhoSO
   cd TrabalhoSO
   ```

2. Dar permissão de execução ao script:
   ```sh
   chmod +x handin_mgr
   ```

3. Executar comandos com a forma:
   ```sh
   ./handin_mgr <comando> [opções] [args]
   ```

> Nota: Não existe modo shell interativo nem opção geral `-h`.  
> Cada comando é chamado diretamente como abaixo.

---

## Estrutura de estado

O programa usa a pasta:

```text
$HOME/.handin_mgr/
```

Com:

- `todos.csv` — ficheiro CSV com as tarefas.
- `runs/` — relatórios de ingestão de entregas.
- `locks/` — diretório usado para locks (exclusão mútua).

Formato das linhas de `todos.csv`:

```text
id;status;prio;due;tags;created;done;title
```

Campos:

- `id` — inteiro positivo incremental.
- `status` — `OPEN` ou `DONE`.
- `prio` — inteiro 1..5 (5 é mais urgente).
- `due` — data `AAAA-MM-DD` ou vazio.
- `tags` — lista separada por vírgulas (ex: `so,tp1,handin`).
- `created` — timestamp `AAAA-MM-DD HH:MM:SS`.
- `done` — timestamp de conclusão ou vazio.
- `title` — título/tarefa em texto livre.

Locks:
- O script usa um lock simples com `mkdir $HOME/.handin_mgr/locks/main.lock` para evitar que dois processos escrevam ao mesmo tempo no `todos.csv`.

---

## Comandos da to‑do list

### 1. `todo-add`

Cria uma nova tarefa na lista.

**Sintaxe:**
```sh
./handin_mgr todo-add "titulo" [-p P] [-d AAAA-MM-DD] [-t tag1,tag2]
```

**Comportamento:**

- `titulo` (obrigatório): primeiro argumento após o comando.
- `-p P` (opcional): prioridade entre 1 e 5.  
  Se não for indicado, a prioridade é `3`.
- `-d AAAA-MM-DD` (opcional): data de prazo (`due`).  
  Se não for indicado, `due` fica vazio (campo em branco).
- `-t tag1,tag2` (opcional): tags separadas por vírgulas.  
  Se não for indicado, `tags` fica vazio.
- Se `todos.csv` não existir, é criado vazio.
- `id` é calculado como `1 + maior id existente` (ou `1` se o ficheiro estiver vazio).
- `created` é preenchido com o timestamp atual.
- A linha é acrescentada a `todos.csv` no formato:
  ```text
  id;OPEN;prio;due;tags;created;;title
  ```
- No fim, o comando imprime:
  ```text
  Tarefa criada com id N
  ```

**Erros:**

- Falta do título → imprime `Erro: Falta o titulo` em `stderr` e termina com código ≠ 0.
- Prioridade fora de 1..5 → `Erro: Prioridade invalida`.
- Opção desconhecida → `Erro: Opcao invalida`.

**Exemplos:**
```sh
./handin_mgr todo-add "Ler enunciado TP1" -p 4 -d 2026-02-20 -t so,tp1
./handin_mgr todo-add "Rever código do handin" -p 5 -t so,tp1,handin
```

---

### 2. `todo-list`

Lista tarefas do `todos.csv`, com filtros e ordenações.

**Sintaxe:**
```sh
./handin_mgr todo-list [-a] [-s sort] [-t tag] [-p minPrio]
```

**Opções:**

- `-a`  
  Mostra tarefas `OPEN` e `DONE`.  
  Sem `-a`, por omissão só mostra `OPEN`.
- `-s sort`  
  Campo de ordenação:
  - `due` — ordena por campo `due` crescente.
  - `prio` — ordena por `prio` decrescente.
  - `created` — ordena por `created` crescente.
- `-t tag`  
  Filtra tarefas que contenham essa tag na coluna `tags`.
- `-p minPrio`  
  Filtra tarefas com prioridade `>= minPrio` (1..5).

**Comportamento:**

1. Se `todos.csv` não existir, o comando termina silenciosamente (sem erro).
2. Copia `todos.csv` para um ficheiro temporário.
3. Se não for usado `-a`, filtra apenas tarefas com `status = OPEN`.
4. Se `-t` for usado, filtra por linhas que contenham a tag indicada.
5. Se `-p` for usado, filtra por tarefas com prioridade >= `minPrio`.
6. Aplica a ordenação pedida com `sort` (se `-s` for usado).
7. Imprime cada linha no formato:
   ```text
   [id] (Pprio) due status tags:tags - title
   ```

**Erros:**

- `minPrio` fora de 1..5 → `Erro: minPrio invalido`.
- `-s` com valor diferente de `due`, `prio` ou `created` → `Erro: sort invalido`.
- Opção desconhecida → `Erro: Opcao invalida`.

**Exemplos:**
```sh
./handin_mgr todo-list
./handin_mgr todo-list -s due
./handin_mgr todo-list -a -p 4
./handin_mgr todo-list -t handin
```

---

### 3. `todo-done`

Marca uma tarefa como concluída (`DONE`).

**Sintaxe:**
```sh
./handin_mgr todo-done <id>
```

**Comportamento:**

- Verifica se o `id` foi passado.
- Verifica se existe uma linha em `todos.csv` que comece com `id;`.
- Se o `status` já for `DONE`, escreve:
  ```text
  Tarefa ja esta DONE
  ```
  e termina sem alterar o ficheiro.
- Caso contrário:
  - Atualiza o `status` para `DONE`.
  - Preenche o campo `done` com o timestamp atual (`AAAA-MM-DD HH:MM:SS`).
  - Mantém os restantes campos iguais.
- No fim, imprime:
  ```text
  Tarefa <id> marcada como DONE
  ```

**Erros:**

- Falta do id → `Erro: Falta o id`.
- `id` inexistente → `Erro: ID inexistente`.

**Exemplos:**
```sh
./handin_mgr todo-done 3
```

---

### 4. `todo-search`

Pesquisa texto no título das tarefas.

**Sintaxe:**
```sh
./handin_mgr todo-search <texto>
```

**Comportamento:**

- Pesquisa case-insensitive (`grep -i`) pelo `<texto>` no ficheiro `todos.csv`.
- Para cada linha encontrada, imprime no mesmo formato de `todo-list`:
  ```text
  [id] (Pprio) due status tags:tags - title
  ```

**Erros:**

- Falta do texto → `Erro: Falta texto de pesquisa`.

**Exemplos:**
```sh
./handin_mgr todo-search "enunciado"
./handin_mgr todo-search "handin"
```

---

## Comandos de gestão de entregas (handin)

As entregas devem respeitar a convenção de nomes:

```text
ALUNO_UC_TP#_YYYYMMDDHHMM
```

Exemplos:
```text
a12345_SO_TP1_202602131845
b54321_SO_TP2_202602101210
```

A estrutura final no repositório de entregas é:

```text
<repo_dir>/<UC>/<TP#>/<ALUNO>/<timestamp>/
```

---

### 5. `handin-ingest`

Lê uma pasta de “inbox” com entregas e copia/move para o repositório organizado.

**Sintaxe:**
```sh
./handin_mgr handin-ingest <inbox_dir> <repo_dir> [-m]
```

**Comportamento:**

- `inbox_dir`: diretório onde estão as entregas (pastas ou ficheiros).
- `repo_dir`: diretório base do repositório de entregas.
- `-m` (opcional, na 4ª posição): se presente, **move** as entregas em vez de copiar.

Passos:

1. Verifica se `<inbox_dir>` e `<repo_dir>` existem.
2. Cria um id de execução:
   ```text
   YYYY-MM-DD_HHMMSS
   ```
3. Cria um relatório em:
   ```text
   $HOME/.handin_mgr/runs/ingest_<timestamp>.txt
   ```
4. Para cada item em `<inbox_dir>`:
   - Valida o nome com a regex  
     `^[a-zA-Z0-9]+_[a-zA-Z0-9]+_TP[0-9]+_[0-9]{12}`
   - Se o nome for inválido:
     ```text
     FAIL;<origem>;nome_invalido
     ```
     é escrito no relatório.
   - Se for válido:
     - Extrai `ALUNO`, `UC`, `TP`, `timestamp`.
     - Cria a pasta destino:
       ```text
       <repo_dir>/<UC>/<TP>/<ALUNO>/<timestamp>/
       ```
     - Copia (`cp -r`) ou move (`mv`) o item para o destino.
     - Em caso de sucesso:
       ```text
       OK;<origem>;<destino>
       ```
     - Em caso de erro de cópia/move:
       ```text
       FAIL;<origem>;erro_copia
       ```

5. No fim, o comando escreve no ecrã:
   ```text
   Relatorio criado em: <caminho_do_relatorio>
   ```

**Limitações atuais:**

- Não marca entregas antigas como `SUPERSEDED`.
- Não descomprime automaticamente ficheiros comprimidos (`.zip`, `.tar`, `.tar.gz`).

**Exemplos:**
```sh
./handin_mgr handin-ingest inbox repo
./handin_mgr handin-ingest inbox repo -m
```

---

### 6. `handin-check`

Valida a estrutura das entregas e produz um relatório CSV com métricas.

**Sintaxe:**
```sh
./handin_mgr handin-check <repo_dir> [-o relatorio.csv]
```

**Comportamento:**

- Percorre recursivamente `<repo_dir>` e identifica diretórios cujo nome seja um timestamp de 12 dígitos (`YYYYMMDDHHMM`).
- A partir do caminho, extrai:
  - `ALUNO` — diretório pai imediato.
  - `TP#` — diretório 2 níveis acima.
  - `UC` — diretório 3 níveis acima.
- Para cada entrega, calcula:
  - `STATUS`: começa em `OK` e passa a `FAIL` se forem encontrados problemas.
  - `problemas`: lista de strings separadas por vírgulas.

**Validações feitas:**

- README:
  - Falta de `README.md` **e** `README.txt` → `missing_readme`.
- `src/`:
  - Falta da pasta `src` → `missing_src`.
  - Pasta `src` vazia (0 ficheiros) → `empty_src`.
- Pastas proibidas:
  - Presença de `node_modules`, `dist` ou `build` → `forbidden_dir`.
- Ficheiros binários:
  - Se algum ficheiro fizer falhar `grep -Iq .` → `binary_found`.
- Ficheiros demasiado grandes:
  - Se algum ficheiro tiver tamanho > 5 MB → `too_large`.

**Métricas:**

- `num_src_files` — nº de ficheiros em `src/`.
- `total_files` — nº total de ficheiros regulares na entrega.
- `total_lines` — total de linhas em todos os ficheiros de texto dentro de `src/`.

**Formato do relatório (uma linha por entrega):**

```text
UC;TP#;ALUNO;timestamp;STATUS;num_src_files;total_files;total_lines;problemas
```

- `STATUS`: `OK` ou `FAIL`.
- `problemas`: vazio ou lista separada por vírgulas (`missing_readme,binary_found,...`).

**Saída:**

- Se for usado `-o relatorio.csv`, escreve o relatório nesse ficheiro.
- Caso contrário, escreve as linhas para `stdout`.

**Integração com a to‑do list:**

- Se `STATUS = FAIL`, o script cria automaticamente uma tarefa em `todos.csv`:
  - `title`:  
    `Corrigir entrega ALUNO UC TP# (problemas)`
  - `tags`:  
    `handin,UC,TP#`
  - `prio`:
    - `5` se `problemas` contiver `missing_src` ou `binary_found`.
    - `4` nos restantes casos.
- Antes de criar, verifica se já existe uma tarefa com o mesmo `title` em `todos.csv` para evitar duplicados.

**Exemplos:**
```sh
./handin_mgr handin-check repo
./handin_mgr handin-check repo -o relatorio_tp1.csv
```

---

### 7. `handin-summary`

Gera um resumo agregado das entregas, a partir da estrutura em `<repo_dir>`.

**Sintaxe:**
```sh
./handin_mgr handin-summary <repo_dir> [-u UC] [-t TP#]
```

**Comportamento:**

1. Percorre `<repo_dir>` e identifica diretórios de timestamp (`YYYYMMDDHHMM`), tal como em `handin-check`.
2. Para cada entrega:
   - Extrai `UC`, `TP#`, `ALUNO`.
   - Aplica filtros se forem dados:
     - `-u UC` → só essa UC.
     - `-t TP#` → só esse TP.
   - Determina um `STATUS` simples:
     - `OK` se existir README (`.md` ou `.txt`) **e** pasta `src/`.
     - `FAIL` caso contrário.
   - Conta:
     - `TOTAL_FILES` — nº total de ficheiros na entrega.
     - `TOTAL_LINES` — linhas totais em ficheiros dentro de `src/`.
   - Guarda uma linha temporária:
     ```text
     UC;TP#;ALUNO;STATUS;TOTAL_FILES;TOTAL_LINES
     ```

3. A partir desse ficheiro temporário calcula:
   - Total de entregas.
   - Quantas `OK` e quantas `FAIL`.
   - Top 5 entregas com mais ficheiros.
   - Top 5 entregas com mais linhas.
   - Lista de alunos com pelo menos uma entrega `FAIL`.

**Formato da saída (texto legível):**

```text
===== RESUMO =====
Total entregas: X
OK: Y
FAIL: Z

===== TOP 5 FICHEIROS =====
UC TP ALUNO - N ficheiros
...

===== TOP 5 LINHAS =====
UC TP ALUNO - M linhas
...

===== ALUNOS COM FAIL =====
aluno1
aluno2
...
```

**Exemplos:**
```sh
./handin_mgr handin-summary repo
./handin_mgr handin-summary repo -u SO -t TP1
```

---

## Limitações e suposições atuais

- O script assume que os nomes das entregas seguem exatamente a convenção  
  `ALUNO_UC_TP#_YYYYMMDDHHMM`. Nomes fora deste padrão são marcados como falha em `handin-ingest`.
- Entregas comprimidas (`.zip`, `.tar`, `.tar.gz`) **não são extraídas automaticamente**; devem ser tratadas manualmente.
- `handin-ingest` não marca entregas antigas como `SUPERSEDED`; mantém todas as versões.
- A deteção de binários é heurística com `grep -Iq`, podendo falhar em casos extremos.
- A criação de tarefas a partir de `handin-check` evita duplicados com base apenas no `title` da tarefa.
- Não há comandos de ajuda (`-h`) por comando; a utilização é a descrita neste README.










## Exemplos de utilização completa

Abaixo está um exemplo de fluxo completo, desde a criação de entregas de teste até ao resumo final e às tarefas geradas automaticamente.

### 1. Preparar diretórios de teste

Dentro da pasta do projeto:

```sh
mkdir -p inbox
mkdir -p repo
```

Criar uma entrega válida:

```sh
mkdir -p inbox/a12345_SO_TP1_202604011200/src
echo "# Trabalho TP1" > inbox/a12345_SO_TP1_202604011200/README.md
echo "int main() { return 0; }" > inbox/a12345_SO_TP1_202604011200/src/main.c
```

Criar uma entrega com problema (sem pasta src):

```sh
mkdir -p inbox/b99999_SO_TP1_202604011300
echo "# Trabalho TP1 sem src" > inbox/b99999_SO_TP1_202604011300/README.md
```

### 2. Ingerir entregas para o repositório

```sh
./handin_mgr handin-ingest inbox repo
```

Saída esperada (exemplo):

```text
Relatorio criado em: /home/<user>/.handin_mgr/runs/ingest_2026-04-01_120000.txt
```

Podemos ver o relatório de ingestão:

```sh
cat ~/.handin_mgr/runs/ingest_*.txt
```

Exemplo de conteúdo:

```text
OK;/caminho/para/inbox/a12345_SO_TP1_202604011200;/caminho/para/repo/SO/TP1/a12345/202604011200
OK;/caminho/para/inbox/b99999_SO_TP1_202604011300;/caminho/para/repo/SO/TP1/b99999/202604011300
```

### 3. Validar entregas e gerar relatório CSV

```sh
./handin_mgr handin-check repo -o relatorio_tp1.csv
```

Ver o ficheiro gerado:

```sh
cat relatorio_tp1.csv
```

Exemplo de linhas:

```text
SO;TP1;a12345;202604011200;OK;1;2;2;
SO;TP1;b99999;202604011300;FAIL;0;1;0;missing_src
```

Neste exemplo:
- A entrega de `a12345` está `OK`.
- A entrega de `b99999` está `FAIL` com o problema `missing_src`.

Como há uma entrega `FAIL`, o `handin-check` cria automaticamente uma tarefa na to‑do list para corrigir essa entrega.

### 4. Ver tarefas geradas automaticamente

Listar tarefas atuais:

```sh
./handin_mgr todo-list
```

Exemplo de saída:

```text
 (P5)  OPEN tags:handin,SO,TP1 - Corrigir entrega b99999 SO TP1 (missing_src)[1]
```

Aqui vemos uma tarefa criada automaticamente com:
- `prio = 5` (erro grave: falta src).
- `tags = handin,SO,TP1`.

### 5. Resumo agregado das entregas

Gerar um resumo global das entregas no repositório:

```sh
./handin_mgr handin-summary repo
```

Possível saída:

```text
===== RESUMO =====
Total entregas: 2
OK: 1
FAIL: 1

===== TOP 5 FICHEIROS =====
SO TP1 a12345 - 2 ficheiros
SO TP1 b99999 - 1 ficheiros

===== TOP 5 LINHAS =====
SO TP1 a12345 - 2 linhas
SO TP1 b99999 - 0 linhas

===== ALUNOS COM FAIL =====
b99999
```

Também é possível filtrar por UC e TP específicos:

```sh
./handin_mgr handin-summary repo -u SO -t TP1
```

Este fluxo mostra como:
1. Criar entregas de exemplo.
2. Ingeri‑las para o repositório.
3. Validar e gerar relatório CSV.
4. Ver tarefas automáticas criadas em caso de falha.
5. Obter um resumo estatístico das entregas.