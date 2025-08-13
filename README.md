# Alexa - Gerador de Frases Aleatórias (SQL)

Subflow para **Node-RED** que seleciona e envia para a Alexa (ou outro nó) uma frase aleatória armazenada em **MariaDB**, com filtros por categoria/estilo e **cache** para evitar repetição. Inclui export do banco e o subfluxo prontos para importar.

---

## 📂 Arquivos neste repositório

- `node_red_db.sql` — **Export do banco** (estrutura +, opcionalmente, dados de exemplo).
- `node-red_subflow.json` — **Subflow do Node-RED** pronto para importar.

---

## 🧭 Como o projeto funciona (visão geral)

1. O subflow recebe parâmetros (categoria, classes, pessoa, saída, echo).
2. Monta uma **query SQL** que:
   - filtra por **categoria** e **classe(s)**;
   - **exclui** frases já usadas recentemente pelo mesmo `node_id` (cache).
3. Executa a query no **MariaDB** e retorna **1 frase aleatória**.
4. Registra a frase escolhida em `cache_frases_temporario`.
5. Envia a frase para a **Alexa** via `notify.alexa_media` (Home Assistant).
6. Se não houver frase disponível (filtros muito restritos), o subflow **limpa o cache do `node_id`** e tenta novamente.

---

## 🛠 Pré-requisitos

- **Node-RED** (3.x+ recomendado).
- **MariaDB/MySQL** acessível pelo Node-RED.
- **Home Assistant** com a integração **Alexa Media Player** (para `notify.alexa_media` e uso do `last_alexa`).
- (Opcional) **Skill/automação própria** para mapear IDs de dispositivos Alexa (útil se quiser “last_alexa” e mapeamentos adicionais).

---

## 1️⃣ Importar o banco (`node_red_db.sql`)

### Via **phpMyAdmin**
1. Criar o banco `node_red_db` (ou usar outro nome).
2. **Importar** → selecionar `node_red_db.sql` → **Executar**.

---

### Permissões mínimas sugeridas para o usuário do Node-RED
```sql
GRANT SELECT, INSERT, DELETE ON node_red_db.* TO 'node_red_user'@'%';
FLUSH PRIVILEGES;
```
> O fluxo só **lê** de `frases` e **grava/apaga** em `cache_frases_temporario`.

---

## 2️⃣ Importar o subflow (`node-red_subflow.json`)

1. Node-RED → **Menu** → **Importar**
2. Selecionar `node-red_subflow.json`
3. Arrastar o subflow **“Alexa Fala Frase SQL”** para o seu fluxo

---

## 🔌 Dependências do Node-RED

Instale (via *Palette Manager* do Node-RED):

- `node-red-node-mysql` (nó **mysql**)

> Reinicie o Node-RED após instalar.


## 3️⃣ Configurar o nó **MySQLdatabase** (dentro do subflow)
Abra o nó **MySQLdatabase** (a conexão usada pelas queries) e configure:

| Campo    | Valor (exemplo) |
|----------|------------------|
| Host     | `core-mariadb`   |
| Port     | `3306`           |
| User     | `node_red_user`  |
| Password | `********`       |
| Database | `node_red_db`    |
| Charset  | `UTF8`           |

> Use o mesmo usuário/senha criados no banco.  
> Se necessário, ajuste **Host** para o endereço/IP do seu servidor MariaDB.  
> Deixe o **Database** `node_red_db` selecionado nas propridades do nó.   
> Selecione e Database `node_red_db` **também** no subfluxo.  

---

## 4️⃣ Personalizar nomes e dispositivos (modelo do subflow)
No **modelo do subflow** (em **Editar modelo de subfluxo**), você pode **editar as opções** que aparecem para o usuário:

- **`pessoa`** → lista de nomes que podem ser mencionados, ex.: `felipe`, `raissa`, `felipe e raíssa`, ` ` *(em branco para nenhum)*
- **`echo`** → IDs/entidades dos dispositivos Alexa, ex.:  
  `media_player.echo_01`, `media_player.echo_02`, `media_player.echo_03`, …, `media_player.casa_toda`, `last_alexa`
- **`normais`**, **`engracadas`**, **`sarcasticas`** → checkboxes padrão (`true` / `false`)
- **`categoria`** → **ID** da categoria (veja a tabela “Categorias” do README)
- **`saida`** → `announce`, `frase` *(Texto)*, `tts`

**Responsabilidade do usuário:** ajustar esses valores para refletir **seus** nomes e **seus** dispositivos.  
Depois de ajustar no modelo, as opções aparecerão no **formulário do subflow** ao usá-lo no fluxo.

---

## 5️⃣ Configurar a integração Alexa (Home Assistant)
- Instale e configure **Alexa Media Player**.  
- Crie/ative o sensor **`last_alexa`** seguindo a documentação da integração.  
- Verifique que as entidades **`media_player.*`** dos seus Echos estão disponíveis (ex.: `media_player.echo_01`).  
- *(Opcional)* Se tiver **skill custom** para obter os IDs nativos dos dispositivos Alexa, você pode incrementar o mapeamento **“last_alexa” → `media_player.*`** no seu fluxo.

---

## 6️⃣ Usando o subflow no fluxo
Ao arrastar o subflow para o fluxo, configure os campos:

- **Categoria**: ex. `Boa Noite` *(ou ID da categoria)*  
- **Pessoa**: `Felipe`, `Raíssa`, `Felipe e Raíssa` ou ` ` *(nenhum)*  
- **Classes**: marque **Normais**, **Engraçadas**, **Sarcásticas** *(uma ou mais)*  
- **Saída**: **Texto** (`frase`), **TTS** (`tts`) ou **Anúncio** (`announce`)  
- **Echo**: escolha o dispositivo (ex.: `media_player.echo_02`) ou `last_alexa`

**Fluxo de execução:**
1. Busca **1 frase aleatória** via SQL (com os filtros).  
2. Verifica **cache** para não repetir (por `node_id` do subflow).  
3. Envia via **`notify.alexa_media`** para o **echo** escolhido, com o tipo de **Saída** selecionado.

---

### ▶️ Exemplos de uso

**Exemplo 1 — Boa noite com sarcasmo**
- Categoria: `3 (Boa Noite)`  
- Classes: `Normal`, `Sarcastica`  
- Pessoa: `Felipe`  
- Saída: `tts`  
- Echo: `media_player.echo_01`

**Exemplo 2 — Boas-vindas em anúncio para a casa toda**
- Categoria: `4 (Boas Vindas)`  
- Classes: `Normal`  
- Pessoa: `Felipe e Raíssa`  
- Saída: `announce`  
- Echo: `media_player.casa_toda`

---

## ⛏️ Dica de manutenção
Limpar cache antigo (ex.: **mais de 7 dias**):
```sql
DELETE FROM cache_frases_temporario
WHERE data_criacao < NOW() - INTERVAL 7 DAY;
```



> Baseado em [mendoncart/alexa-frases-aleatorias](https://github.com/mendoncart/alexa-frases-aleatorias).
> Esta variante usa **MariaDB/SQL** e **cache por `node_id`**.

