---
title: 'Como deixar o Swagger com tema dark mode usando Swaggo e Golang'
date: 2023-11-11T15:39:40-03:00
draft: false
tableOfContents: false
---

Recentemente criei um port mostrando como deixar o Swagger com tema dark mode utilizando [NestJS](posts/swagger-dark-nestjs/), agora vou mostrar como deixar o Swagger em dark mode utilizando o [Swaggo](https://github.com/swaggo/swag) com Go.

## O que é o Swaggo?

O Swaggo é uma ferramenta que nos ajuda a documentar nossa API desenvolvida em [GO](https://go.dev/), gerando a documentação no padrão da [OpenAPI](https://www.openapis.org/).

Não vamos focar em como utilizar o Swaggo, mas vou deixar aqui um excelente post no [Dev.to](https://dev.to/booscaaa/documentanto-uma-api-go-com-swagger-2k05) que ensina como fazer isso.

## Criando o projeto de exemplo

Vamos criar um exemplo mais alinhado com uma utilização no mundo real, para isso vamos utilizar o [Go Chi](https://go-chi.io/), que é o roteador muito simples, mas que facilita muito na criação de rotas em Go.

Vamos iniciar o projeto rodando o comando:

```go
  go mod init swagger-dark-mode
```

Isso vai criar nosso arquivo `go.mod`, que vai servir para gerenciar nossos pacotes.

## Organizando o projeto

![Project structure](fhvc908cxavb09bn0.png)

Vamos organizar seguindo um padrão muito utilizado pela comunidade do Go, você pode ver nesse [repositório](https://github.com/golang-standards/project-layout)

- **cmd**: [Aqui](https://github.com/golang-standards/project-layout/tree/master/cmd) é onde vamos deixar nosso arquivos que iniciam a nossa aplicação.
  - **webserver**: Aqui é onde vamos deixar o `main.go` que inicia nosso webserver.
- **internal**: [Nessa pasta](https://github.com/golang-standards/project-layout/tree/master/internal) onde deve ficar todo o código da nossa aplicação.
  - **handler**: Aqui vai ficar os arquivos responsáveis por receber nossas solicitações http, você pode conhecer também como controllers.
    - **routes**: Aqui vamos organizar nossas rotas, incluido a rota da nossa documentação.

`main.go`:

```go
  package main

  import (
    "fmt"
    "net/http"
    "swagger-dark-mode/internal/handler/routes"

    "github.com/go-chi/chi/v5"
  )

  func main() {
    r := chi.NewRouter()
    routes.InitRoutes(r)

    fmt.Println("Server running on port 8080")
    http.ListenAndServe(":8080", r)
  }
```

No arquivo `main.go`, iniciamos nosso router com go chi, iniciamos nossas rotas com `routes.InitRoutes()`, e damos start em nosso server que vai rodar na porta `8080`

Você pode baixar os pacotes do go chi e swaggo com o comando:

```bash
  go get github.com/swaggo/http-swagger github.com/go-chi/chi/v5
```

É necessário instalar o swag na sua máquina, veja como neste [link](https://github.com/swaggo/swag#getting-started)

`routes.go`:

```go
  package routes

  import (
    "swagger-dark-mode/internal/handler"

    _ "swagger-dark-mode/docs"

    "github.com/go-chi/chi/v5"
    httpSwagger "github.com/swaggo/http-swagger"
  )

  var (
    docsURL = "http://localhost:8080/docs/doc.json"
  )

  //	@title		Swagger Dark Mode
  //	@version	1.0
  func InitRoutes(r chi.Router) {
    r.Get("/docs/*", httpSwagger.Handler(httpSwagger.URL(docsURL)))

    r.Get("/user", handler.GetUser)
  }
```

No arquivo `routes.go` é onde criamos nossas rotas, criamos uma rota do tipo GET `/user` que chama nosso handler `GetUser` e outra rota
GET para nossa docs `/docs/*` o `/*` após o path `docs`, indica que qualquer combinação de caracteres pode aparecer na posição correspondente.

o import `"_ swagger-dark-mode/docs"` vem da pasta gerada após iniciar o swag, usando o comando:

```bash
  swag init -g internal/handler/routes/routes.go
```

É necessário rodar esse comando sempre que houver alterações na sua documentação, o swag vai criar um pasta chamada docs, você não precisa alterar nada nessa pasta, veja como fica agora a estrutura do projeto:

![Project structure](9sd8fF45GHjgfgQ.png)

Dentro do `swagger.json` fica o arquivo no padrão da OpenAPI.

o caminho `internal/handler/routes/routes.go` deve ser onde está as anotações gerais do swag, no nosso caso setamos apenas o `@title` e o `@version`, veja todas as anotações possíveis [aqui](https://github.com/swaggo/swag#general-api-info).

`user.go`:

```go
  package handler

  import (
    "encoding/json"
    "log/slog"
    "net/http"
  )

  type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
  }

  // Get fake user
  //	@Summary		 Get Fake user
  //	@Description Get Fake user for example
  //	@Tags			   user
  //	@Accept			 json
  //	@Produce		 json
  //	@Success		 200	{object}	User
  //	@Failure		 500
  //	@Router			 /user [get]
  func GetUser(w http.ResponseWriter, r *http.Request) {
    user := User{
      ID:    1,
      Name:  "John Doe",
      Email: "jonh.doe@email.com",
    }

    userMarshal, err := json.Marshal(user)
    if err != nil {
      slog.Error("Error marshalling user", err)
      http.Error(w, err.Error(), http.StatusInternalServerError)
      return
    }

    w.Write(userMarshal)
  }
```

Se desejar formatar suas anotações do swag basta rodar o comando:

```bash
  swag fmt
```

No arquivo `user.go` é onde podemos validar nossa requisição, onde podemos pegar o body por exemplo e transformar em um [struct go](https://go.dev/ref/spec#Struct_types). Criamos um user fake, fizemos o enconding da a struct `User` para Json usando [Marshal](https://pkg.go.dev/encoding/json#Marshal), caso não aconteça um erro duranto o Marshal, retornamos o json usando o `w.writer`.

## Rodando o projeto

Tudo pronto para rodar o projeto, caso não tenha instalado os pacotes, rode o comando:

```bash
  go mod tidy
```

Esse comando vai no arquivo `go.mod` e faz o download das dependências.

Vamos iniciar o projeto, rodando nosso `main.go`

```bash
  go run cmd/webserver/main.go
```

Se tudo estiver correto, vamos ter no terminal a mensagem _Server running on port 8080_.

Acessando nossa rota via `http://localhost:8080/user` teremos esse resultado:

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "jonh.doe@email.com"
}
```

Acessando nossa outra rota `http://localhost:8080/docs/index.html`, teremos nossa documentação com swagger:

![Swaggo light](sopfdf832nmf393n339RfgdfYH.png)

![Meme my eyes](https://media4.giphy.com/media/84BjZMVEX3aRG/giphy.gif?cid=7941fdc6zrsyrot6atdj7as4jvaiplgkjqkf9188qmmz57u0&ep=v1_gifs_search&rid=giphy.gif&ct=g)

## Criando nosso CSS e JS

Bom, finalmente vamos ao intuito do post, deixar o tema do swagger em dark mode, infelizmente não existe nada nativo ou que seja simples quanto deixar o swagger em dark mode utilizando o [NestJS](https://nestjs.com/). _(Pelo menos até a data de pulbicação desse post)_.

Vamos precisar injetar nosso css customizado no swaggo, para isso você pode utilizar o css desse [gist](https://gist.github.com/wiliamvj/674cbdf6de5377780013e6ff2e5667c3), por´m também não conseguimos injetar o css diretamente, mas conseguimos injetar um JavaScript, com isso também se torna possível manipular a [DOM](https://www.w3schools.com/js/js_htmldom.asp) e consequentemente injetar css.

Vamos criar uma pasta dentro da pasta `docs`, chamado `custom`, onde vamos colocar nossas customizações.

_Você pode criar fora da pasta `docs`, já que é uma pasta gerada dinamicamente pelo swaggo, pode ser substituida e acabar perdendo sua customização_, mas para este exemplo vamos deixar dentro da pasta `docs` mesmo.

Dentro do custom vamos criar 2 arquivos, `custom_css.go` e `custom_layout.go`.

`custom_css.go`:

```go
  package custom

  var customCSS = `css do gist`
```

No arquivo `custom_css.go`, apenas retornamos o css que deixe no [gist](https://gist.github.com/wiliamvj/674cbdf6de5377780013e6ff2e5667c3) em string.

`custom_layout.go`:

```go
  package custom

  import "fmt"

  var CustomLayoutJS = fmt.Sprintf(`
      // dark mode
      const style = document.createElement('style');
      style.innerHTML = %s;
      document.head.appendChild(style);
    `, "`"+customCSS+"`")
```

No arquivo `custom_layout.go` criamos noss JavaScript para ser injetado em nosso swagger, criamos uma tag style e adicionamos ao DOM, convertamos em string utilizando o `Sprintf` do pacote [fmt](https://pkg.go.dev/fmt), veja um post meu sobre o pacote fmt [aqui](/posts/fmt-go/).

`""+customCSS+""`, isso envolve o arquivo JS em aspas duplas.

## Aplicando o Dark mode

Vamos finalmente aplicar nosso dark mode, para isso vamos alterar no arquivo `routes.go`:

```go
  func InitRoutes(r chi.Router) {
    r.Get("/docs/*", httpSwagger.Handler(httpSwagger.URL(docsURL),
      httpSwagger.AfterScript(custom.CustomJS),
      httpSwagger.DocExpansion("none"),
      httpSwagger.UIConfig(map[string]string{
      "defaultModelsExpandDepth": `"-1"`,
      }),
    ))

    r.Get("/user", handler.GetUser)
  }
```

`httpSwagger.AfterScript(custom.CustomJS)`: Isso injeta nosso JS no swagger depois da página ser carregada.
`httpSwagger.DocExpansion("none")`: Isso faz com que cada o endpoint abre expandido ou não (gosto pessoal), mas ajuda quando sua documentação possui muitos endpoints, no exemplo o `-1` faz com que por padrão não fique expandido as rotas.
`"defaultModelsExpandDepth": "-1"`: Isso faz com que os models sejam ocultos, caso precise deixar visível basta remover.

Existe mais opções possíveis nas [docs](https://swagger.io/docs/open-source-tools/swagger-ui/usage/configuration/) do swagger.

Agora rodando novamente nosso projeto, já teremos o swagger em dark mode:
![Swaggo dark mode](0s9g7ansY45Joi.png)

![Meme liked](https://media.giphy.com/media/m2Q7FEc0bEr4I/giphy.gif)

## Personalizando ainda mais

Agora com o poder da manipulação da DOM e com paciência, podemos modificar o layout da forma que desejarmos, vamos por exemplo altera a logo e favicon e o title.

Vamos modificar nosso `custom_layout.go`

```go
  var CustomJS = fmt.Sprintf(`
    // set custom title
      document.title = 'Swagger Dark Mode With Go';

      // set custom favicon
      const link = document.createElement('link');
      link.rel = 'icon';
      link.type = 'image/x-icon';
      link.href = 'data:image/png;base64,%s';
      document.head.appendChild(link);

      // set custom logo
      const image = document.querySelector('.link img');
      const base64URL = 'data:image/png;base64,%s';
      image.src = base64URL;

      // dark mode
      const style = document.createElement('style');
      style.innerHTML = %s;
      document.head.appendChild(style);
    `, CustomLogo, CustomLogo, "`"+customCSS+"`")
```

Adicionei um title, favicon e logo, usando um base64 para as imagens. Salvamos esse base64 em um arquivo chamado `images.go` dentro da pasta `custom`.

```go
  package custom

  var (
    CustomLogo = `base64 aqui`
  )
```

Apenas como exemplo, o favicon e a logo foram utilizados a mesma imagem em base64, mas você poderia separar, veja como ficou:
![Swaggo dar mode](fg89Peer124gfh567ghjJK78.png)

## Considerações finais

Como podemos ver, não é tão complicado personalizar, apesar da personalização ser um pouco trabalhosa e parecer não ser a mais adequeada, ainda conseguimos deixar o tema do swagger com uma aparência mais agradável para quem for consumir nossa api.

## Link do repositório

[repositório](https://github.com/wiliamvj/swagger-dark-mode) do projeto
