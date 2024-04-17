---
title: 'Usando migrations com Golang'
date: 2023-11-18T17:00:30-03:00
draft: false
tableOfContents: false
thumbnail: 'dsfU80Olpdrfdg.png'
---

![thumbnail](dsfU80Olpdrfdg.png)

## O que são migrations?

As migrations são muito utilizadas para manter um controle de versão da estrutura do seu banco de dados, é uma solução muito útil para manter seu banco de dados organizado.

Imagine que você tenha uma tabela de usuários e precise inserir um novo campo nessa tabela, sem migrations você precisaria rodar um SQL na mão, como por exemplo:

```sql
  ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(255) NULL;
```

Agora, toda vez que precisar recriar a tabela `users`, você vai precisar lembrar de criar este campo, a menos que alterar a criação original da tabela `users`, porém isso começa a fica inviável a medida que sua tabela e sua aplicação crescem, por isso utilizar as migrations são uma ótima opção.

## As migrations

O funcionamento das migrations são relativamente simples, geralmente temos um arquivo de `up` e outro de `down`, alguns ORMs como o [PrismaORM](https://www.prisma.io/docs/concepts/components/prisma-migrate) criam apenas 1 arquivo, no arquivo de `up` criamos nosso SQL que vai criar ou alterar nosso banco, no arquivo de `down` criamos o SQL que desfaz a alteração.

### Qual a vantagem?

Agora com esses arquivos, mantemos um histórico de alterações do banco de dados, cada alteração tem seu arquivo de `up` e `down`, agora se precisarmos criar as tabelas, rodamos todos os arquivo de `up` e tudo é criado, se precisar reverter, basta rodar o `down`.

## Migrations em Go

O Go não oferece nativamente suporte ao uso de migrations, mas poderíamos utilizar o ORM que tenha essa funcionalidade, como o [GORM](https://gorm.io/docs/migration.html) que é o mais utilizado pela comunidade, mas podemos utilizar as migrations sem o uso de um ORM, para isso vamos utilizar o pacote [golang-migrate](https://github.com/golang-migrate/migrate).

## Pacote Golang Migrate

O pacote [golang-migrate](https://github.com/golang-migrate/migrate) é o mais recomendado para isso, já temos tudo que precisamos para gerenciar nossas migrations e oferece suporte a praticamente todos os bancos de dados, para o nosso exemplo vamos utilizar o PostgreSQL.

## Nosso projeto de exemplo

Já criei previamente um [projeto](https://github.com/wiliamvj/golang-migrations) simples, mas vou explicar rapidamente, pois o foco é utilizar as migrations.

![Project structure](F6ghjghj23dfgfgjhkUo.png)

Teremos essa estrutura, bem simples, o código para exemplo deve ficar inteiro em `main.go`:

```go
  package main

  import (
    "database/sql"
    "fmt"
    "log"
    "os"

    "github.com/joho/godotenv"
    _ "github.com/lib/pq"
  )

  func main() {
    // load .env file
    godotenv.Load()

    postgresURI := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", postgresURI)
    if err != nil {
      log.Panic(err)
    }
    err = db.Ping()
    if err != nil {
      db.Close()
      log.Panic(err)
    }

    fmt.Println("Connected to database")

    // keep the program running
    select {}
  }
```

## Utilizando o golang-migrate

Precisamos instalar a CLI do pacote golang-migrate, veja como instalar [aqui](https://github.com/golang-migrate/migrate/tree/master/cmd/migrate), rode o comando:

```bash
  migrate -version
```

Se a saida for algo como:

```bash
  v4.16.2
```

Todo certo para continuar! O próximo passo é criar nossa primeira migrations com o comando:

```bash
  migrate create -ext=sql -dir=internal/database/migrations -seq init
```

- `ext`: determina a extensão, vamos usar o sql.
- `dir`: Aqui fica o diretório onde vai ser criado as nossas migrations.
- `seq`: Determina a sequência do nome do arquivo da migrations, vamos usar numérico, pode ser usado timestamp.

Com isso, você vai perceber que foi criado uma pasta chamada **migrations** dentro da pasta **database**.

![Project Migrations](mrt8kjHd62iPdg.png)

Foi criado o arquivo `up` e o arquivo `down`, na sequência, como é a primeira fica _000001_, se rodar novamente o comando `migrate create`, vai criar a migration _000002_. Agora vamos criar nosso SQL e rodar as migrations:

No arquivo de `up`, vamos criar a seguinte tabela:

```sql
  CREATE TABLE users (
    id VARCHAR(36) NOT NULL PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP(3) NOT NULL
  );
```

Agora no arquivo de `down` vamos remover a tabela:

```sql
  DROP TABLE IF EXISTS users;
```

Com nosso SQL pronto, podemos rodar o `up` da nossa migrations, não se esqueça de garantir que seu banco esteja rodando, para isso deixei no projeto um arquivo docker compose para rodar uma imagem do PostgreSQL.

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose up
```

- `path`: Informa onde estão as nossas migrations.
- `database`: Url da conexão com o banco de dados.
- `-verbose`: Apenas para exibir todas as execuções.

Se acessarmos algum cliente como o [PgAdmin](https://www.pgadmin.org/) ou [Beekeeper](https://www.beekeeperstudio.io/), ou acessando seu container via bash e verificando via CLI, poderemos ver que a tabela foi criada com sucesso:

![Beekeeper](gfasoijfFG567fgghj.png)

Agora podemos rodar o `down`, é exatamente o mesmo comando, porém alterando de `up` para `down`:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose down
```

Com isso a tabela é removida.

### Adicionando mais campos

Vamos ver agora como ficaria se fosse necessário adicionar mais um campo na tabela `users`, sem migrations, teríamos que alterar diretamente a tabela original, mas com migrations não precisamos, vamos criar outra migration:

```bash
  migrate create -ext=sql -dir=internal/database/migrations -seq init
```

Vai criar a migration `up` e `down` com sequencial _000002_, vamos adicionar o campo `phone`:

`up`:

```sql
  ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(255) NULL;
```

`down`:

```sql
  ALTER TABLE "users" DROP COLUMN "phone";
```

Ao rodar novamente:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose up
```

Nosso campo `phone` é adicionado a tabela user, mas e se eu quiser fazer o `down` apenas na migration que adiciona o campo `phone`? É possível, basta usar o mesmo comando, passando o valor 1, que significa que deseja desfazer a última migration:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose down 1
```

O campo `phone` é removido.

## Facilitando o uso da CLI

Como você pode perceber, os comandos do golang-migrate podem ser um pouco cansativos de usar, podemos facilitar usando um arquivo [makefile](https://dev.to/antoniomotta/makefile-22nh).

```yaml
  include .env

  create_migration:
    migrate create -ext=sql -dir=internal/database/migrations -seq init

  migrate_up:
    migrate -path=internal/database/migrations -database "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}?sslmode=disable" -verbose up

  migrate_down:
    migrate -path=internal/database/migrations -database "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}?sslmode=disable" -verbose down

  .PHONY: create_migration migrate_up migrate_down
```

Criamos atalhos que rodam o comando de `migrate`, precisamos incluir nossas envs usando o `include .env`, depois criamos os comandos:

- `create_migration`: Cria nossos arquivos de migration.
- `migrate_up`: Executa nossas migrations `up`.
- `migrate_down`: Executa nossas migrations `down`.
- `PHONY`: Garante que vai executar um comando, o makefile pode tentar pegar um arquivo, caso existe arquivos com o nome `migrate_up` por exemplo.

Com isso, basta usar o comando:

```bash
  make create_migration
```

E teremos nossos arquivos de migration criados, isso vale para os demais atalhos criados.

## Considerações finais

Nesse post vimos como utilizar as migrations e o quanto elas são importantes para manter um histórico de alterações e facilitar a manutenção do nosso banco de dados. Mas vale ressaltar que o uso de forma errada das migrations podem acabar fazendo você perder os dados do seu banco, por isso é importante ter esse conhecimento de como funcionam as migrations e como seu ORM ou sua linguagem lidam com as migrations.

Evite apagar migrations, elas podem se tornar um pesadelo se não forem utilizadas de forma correta.

## Link do repositório

[repositório](https://github.com/wiliamvj/golang-migrations) do projeto

[Gopher credits](https://github.com/MariaLetta/free-gophers-pack)
