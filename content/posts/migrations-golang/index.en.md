---
title: "Using migrations with Golang"
date: 2023-11-18T17:00:30-03:00
draft: false
tableOfContents: false
thumbnail: "dsfU80Olpdrfdg.png"
---

![thumbnail](dsfU80Olpdrfdg.png)

## What are migrations?

Migrations are widely used to maintain version control of your database structure, it is a very useful solution for keeping your database organized.

Imagine that you have a users table and need to insert a new field into that table, without migrations you would need to run SQL by hand, for example:

```sql
  ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(255) NULL;
```

Now, every time you need to recreate the `users` table, you will need to remember to create this field, unless you change the original creation of the `users` table, but this becomes unfeasible as your table and application grow , so using migrations is a great option.

## The migrations

The way migrations work is relatively simple, we generally have an `up` file and another `down` file, some ORMs like [PrismaORM](https://www.prisma.io/docs/concepts/components/prisma-migrate) create only 1 file, in the `up` file we create our SQL that will create or change our database, in the `down` file we create the SQL that undoes the change.

### What's the advantage?

Now with these files, we maintain a history of database changes, each change has its `up` and `down` file, now if we need to create the tables, we run all the `up` files and everything is created, if If you need to revert, just run `down`.

## Migrations in Go

Go does not natively support the use of migrations, but we could use the ORM that has this functionality, such as [GORM](https://gorm.io/docs/migration.html) which is the most used by the community, but We can use migrations without using an ORM, for this we will use the [golang-migrate](https://github.com/golang-migrate/migrate) package.

## Golang Migrate Package

The [golang-migrate](https://github.com/golang-migrate/migrate) package is the most recommended for this, we already have everything we need to manage our migrations and it offers support for practically all databases, so In our example we will use PostgreSQL.

## Example project

I have previously created a simple [project](https://github.com/wiliamvj/golang-migrations), but I will explain it quickly, as the focus is on using migrations.

![Project structure](F6ghjghj23dfgfgjhkUo.png)

We will have this structure, very simple, the code for the example should be in `main.go`:

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

## Using golang-migrate

We need to install the CLI of the golang-migrate package, see how to install [here](https://github.com/golang-migrate/migrate/tree/master/cmd/migrate), run the command:

```bash
  migrate -version
```

If the output is something like:

```bash
  v4.16.2
```

All right to continue! The next step is to create our first migrations with the command:

```bash
  migrate create -ext=sql -dir=internal/database/migrations -seq init
```

- `ext`: defines the extension, let's use sql.
- `dir`: here is the directory where our migrations will be created.
- `seq`: defines the sequence of the migrations file name, we will use numeric, timestamp can be used.

With this, you will notice that a folder called **migrations** was created inside the **database** folder.

![Project Migrations](mrt8kjHd62iPdg.png)

The `up` file and the `down` file were created, in sequence, as the first is _000001_, if you run the `migrate create` command again, it will create the migration _000002_. Now let's create our SQL and run the migrations:

In the `up` file, we will create the following table:

```sql
  CREATE TABLE users (
    id CHAR(36) NOT NULL PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP(3) NOT NULL
  );
```

Now in the `down` file we will remove the table:

```sql
  DROP TABLE IF EXISTS users;
```

With our SQL ready, we can run the `up` of our migrations, don't forget to ensure that your database is running, for this I left a docker compose file in the project to run a PostgreSQL image.

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose up
```

- `path`: Defines where our migrations are..
- `database`: Database connection url.
- `-verbose`: Just to display all runs.

If we access a client like [PgAdmin](https://www.pgadmin.org/) or [Beekeeper](https://www.beekeeperstudio.io/), or accessing its container via bash and checking via CLI, we can see that the table was created successfully:

![Beekeeper](gfasoijfFG567fgghj.png)

Now we can run `down`, it's exactly the same command, but changing from `up` to `down`:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose down
```

With this the table is removed.

### Adding more fields

Let's see now what it would look like if it were necessary to add another field to the `users` table, without migrations, we would have to directly change the original table, but with migrations we don't need to, let's create another migration:

```bash
  migrate create -ext=sql -dir=internal/database/migrations -seq init
```

It will create the `up` and `down` migration with sequential _000002_, let's add the `phone` field:

`up`:

```sql
  ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(255) NULL;
```

`down`:

```sql
  ALTER TABLE "users" DROP COLUMN "phone";
```

When running again:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose up
```

Our `phone` field is added to the user table, but what if I want to `down` only in the migration that adds the `phone` field? It's possible, just use the same command, passing the value 1, which means you want to undo the last migration:

```bash
  migrate -path=internal/database/migrations -database "postgresql://golang_migrate:golang_migrate@localhost:5432/golang_migrate?sslmode=disable" -verbose down 1
```

The `phone` field is removed.

## Making the CLI easier to use

As you can see, the golang-migrate commands can be a little tiring to use, we can make it easier by using a file [makefile](https://dev.to/antoniomotta/makefile-22nh).

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

We create shortcuts that run the `migrate` command, we need to include our envs using `include .env`, then we create the commands:

- `create_migration`: Creates our migration files.
- `migrate_up`: Run migrations `up`.
- `migrate_down`:Run migrations `down`.
- `PHONY`: It will execute a command, the makefile may try to get a file, if there are files with the name `migrate_up` for example.

With that, just use the command:

```bash
  make create_migration
```

And we will have our migration files created, this applies to the other shortcuts created.

## Final considerations

In this post we saw how to use migrations and how important they are to maintain a history of changes and facilitate the maintenance of our database. But it is worth mentioning that using migrations incorrectly can end up causing you to lose data from your database, so it is important to have this knowledge of how migrations work and how your ORM or language deals with migrations.

Avoid deleting migrations, they can become a nightmare if not used correctly.

## Repository link

[repository](https://github.com/wiliamvj/golang-migrations) project

[Gopher credits](https://github.com/MariaLetta/free-gophers-pack)
