# Alembic

- Pode trabalhar em toda a camada de DDL (Data Definition Language)
- Fornece scripts de migração de schemas para upgrades e downgrades
- Suporte a geração de SQL (offline)
- API minimalista

## Instalação:

```bash
pip install alembic
```

## Inicializando o Alembic

```bash
alembic init <nome> # por padrão migrations: alembic init migrations
```

## Configurações gerais

Dentro do arquivo `alembic.ini`, tem as configurações de onde serão salvas as migrações, o path e a URI do banco de dados.

Dentro do arquivo `env.py` temos os arquivos para gerar as migrações.

### Gerando uma migração

```bash
alembic revision -m "mensagem de migração"
# a mensagem passada identificará o arquivo de migração (parecido com commit)
```

Dentro do arquivo gerado pela migração, teremos duas funções principais que são "inversas": `upgrade()` e `downgrade()`

### API de operações

A API de operações funciona para os comandos DDL:

- Criação de tabelas (`CREATE TABLE`)
- Alteração de tabelas (`ALTER TABLE`)
- Deleção de tabelas (`DROP TABLE`)

Essas operações serão feitas dentro das funções `upgrade()` e `downgrade()`

### Configuração do banco de dados no alembic

Antes de aplicar uma migração de fato, devemos dizer ao Alembic onde está nosso banco de dados. Para isso, precisamos passar a URI do banco no `alembic.ini`

```toml
[alembic]
sqlalchemy.url = sqlite:///database.db
```

Entretanto, podemos ter a configuração do banco de dados feito direto no `env.py`:

```python
from alembic import context 
from api.settings import get_settings

settings = get_settings()

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
```

Para subir as migrações no banco de dados, basta usarmos o seguinte comando:

```bash
alembic upgrade head
```

Caso quisermos fazer um downgrade para a última versão da migração, usamos o seguinte comando:

```bash
alembic downgrade -1
```

### Desfazendo e aplicando migrações

O alembic possui uma boa funcionalidade de histórico de migrações e como progredir de uma migração para outra:

```bash
alembic history
alembic history --indicate-current # ou só -i (mostra onde estou)

alembic upgrade <ID>
alembic upgrade head
alembic upgrade +1

alembic downgrade <ID>
alembic downgrade base
alembic downgrade -1
```

## SQLA CodeGen

Um cenário comum é termos um banco existente ou ter começado um projeto sem modelos evolutivos. Nesse caso, podemos fazer um "reflection" do banco de dados em código.

- Instalação: `pip install sqlacodegen`

Usando a URI do banco, o codegen vai construir os modelos usando SQLAlchemy ORM:

```bash
# sqlacodegen <DB_URI>
sqlacodegen sqlite:///database.db
```

## Migrações automáticas

Dentro do arquivo `env.py`, precisamos passar o Base.metadata para o `target_metadata`:

```python
from core.database import Base

target_metadata = Base.metadata
```

Lembrando que para esse caso, caso tenhamos outros modelos, precisamos importar eles também, para que o `Base.metadata` registre eles também:

```python
from core.database import Base
from modules.users.models import User
from modules.posts.models import Post

target_metadata = Base.metadata

# podemos criar um arquivo para jogar todos os models lá e trazemos todos de uma vez, exemplo: registry.py com todos imports dos models
```

Agora, podemos fazer uma migração automática com o alembic:

```bash
alembic revision --autogenerate -m "tabelas users e posts"
```

## Alguns problemas que podemos ter

### Sem acesso ao banco de produção

Por questões de segurança, as vezes não podemos aplicar migração no banco de produção. Nesse caso, podemos fazer uma migração offline para gerar um código SQL:

```bash
alembic upgrade +1 --sql
```

Para downgrades, precisamos passar o **ID** da versão que estou para a versão que vamos voltar:

```bash
alembic downgrade <id_current>:<id_past> --sql
# podemos usar downgrade head:-1 também, exemplo:
# alembic downgrade head:-1 --sql
```
