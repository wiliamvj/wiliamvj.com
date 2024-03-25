---
title: 'API completa em Golang - Parte 5'
date: 2024-01-06T19:03:34-03:00
draft: false
tableOfContents: false
---

![thumbnail](uidGhfFHhkn90ndftqcvkFfgFasSd.png)

## O que vamos fazer?

Na parte 5 do nosso crud, vamos fazer a autenticação do usuário, criando um endpoint de login que retorna um JWT, vamos proteger as rotas, impedindo o uso sem token, vamos criar uma função que recebe o token e devolve uma estrutura com os dados salvos no token e ainda vamos adicionar um middleware de logs que adiciona dados do usuário que fez a chamada a nossa api.

Se ainda não viu os posts anteriores leia eles primeiro.

[parte 1](posts/api-golang-parte-1/) |
[parte 2](posts/api-golang-parte-2/) |
[parte 3](posts/api-golang-parte-3/) |
[parte 4](posts/api-golang-parte-4/) |

## Gerando o JWT

Não vou me aprofundar em como o JWT funciona, para não ficar um post muito extenso, mas você pode ler esse [post](https://dev.to/gabrielhsilvestre/o-basico-jwt-json-web-token-2akc) e entender melhor.

Para conseguirmos gerar nosso JWT precisamos ajustar algumas coisas, primeiro vamos adicionar nas variáveis de ambiente o segredo do nosso jwt e o tempo de expiração:

`.env`:

```yaml
## JWT
JWT_SECRET=secret
JWT_EXPIRES_IN=60000
```

Para teste deixaremos esses valores, mas nunca compartilhe o segredo do seu jwt e utilize uma segredo seguro. `JWT_EXPIRES_IN=6000` determina o tempo que o nosso token será válido, deixei apenas 1 minuto.

Agora precisamos configurar nosso arquivo do viper que faz a importação das envs `env.go`:

```go
  type config struct {
    GoEnv        string `mapstructure:"GO_ENV"`
    GoPort       string `mapstructure:"GO_PORT"`
    DatabaseURL  string `mapstructure:"DATABASE_URL"`
    ViaCepURL    string `mapstructure:"VIA_CEP_URL"`
    JwtSecret    string `mapstructure:"JWT_SECRET"`
    JwtExpiresIn int    `mapstructure:"JWT_EXPIRES_IN"`
    TokenAuth    *jwtauth.JWTAuth
  }
```

Adicionamos as opções `JwtSecret`, `JwtExpiresIn` e o `TokenAuth` que importamos do pacote [jwtauth](github.com/go-chi/jwtauth) do go-chi, o `TokenAuth` vai ser responsável por lidar com a geração dos tokens.

Depois disto precisamos inicializar o `TokenAuth`, vamos fazer isso também no arquivo `env.go` antes do nosso `return`:

```go
  Env.TokenAuth = jwtauth.New("HS256", []byte(Env.JwtSecret), nil)
  return Env, nil
```

Definimos que o nosso algoritmo de assinatura será o "HS256" para gerar tokens JWT. O mais utilizado é o `HS256`, porém existem outros como:

```go
	ES256       SignatureAlgorithm = "ES256" // ECDSA using P-256 and SHA-256
	ES384       SignatureAlgorithm = "ES384" // ECDSA using P-384 and SHA-384
	ES512       SignatureAlgorithm = "ES512" // ECDSA using P-521 and SHA-512
	EdDSA       SignatureAlgorithm = "EdDSA" // EdDSA signature algorithms
	HS256       SignatureAlgorithm = "HS256" // HMAC using SHA-256
	HS384       SignatureAlgorithm = "HS384" // HMAC using SHA-384
	HS512       SignatureAlgorithm = "HS512" // HMAC using SHA-512
	NoSignature SignatureAlgorithm = "none"
	PS256       SignatureAlgorithm = "PS256" // RSASSA-PSS using SHA256 and MGF1-SHA256
	PS384       SignatureAlgorithm = "PS384" // RSASSA-PSS using SHA384 and MGF1-SHA384
	PS512       SignatureAlgorithm = "PS512" // RSASSA-PSS using SHA512 and MGF1-SHA512
	RS256       SignatureAlgorithm = "RS256" // RSASSA-PKCS-v1.5 using SHA-256
	RS384       SignatureAlgorithm = "RS384" // RSASSA-PKCS-v1.5 using SHA-384
	RS512       SignatureAlgorithm = "RS512" // RSASSA-PKCS-v1.5 using SHA-512
```

### Criando nosso dto e response

Precisamos de um novo dto, dentro do `user_dto.go` vamos criar um dto chamado `LoginDTO`:

```go
  type LoginDTO struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8,max=40"`
  }
```

Com algumas validações que já abordamos na [parte 3](https://wiliamvj.com/posts/api-golang-parte-3/#iniciando-os-dtos)

Também precisamos criar uma response, para retornar o token gerado, para isso dentro do **handler/response** no arquivo `user_response.go`, vamos criar o `UserAuthToken`:

```go
  type UserAuthToken struct {
    AccessToken string `json:"access_token"`
  }
```

### Criando nosso service

Para o nosso service, vamos separar essa parte de autenticação em outro arquivo, para isso crie dentro da pasta **service/userservice** um arquivo chamado `auth_service.go`.

Primeiro vamos criar nossa interface, vamos chamar de `Login`:

```go
  type UserService interface {
    CreateUser(ctx context.Context, u dto.CreateUserDto) error
    UpdateUser(ctx context.Context, u dto.UpdateUserDto, id string) error
    GetUserByID(ctx context.Context, id string) (*response.UserResponse, error)
    DeleteUser(ctx context.Context, id string) error
    FindManyUsers(ctx context.Context) (*response.ManyUsersResponse, error)
    UpdateUserPassword(ctx context.Context, u *dto.UpdateUserPasswordDto, id string) error
    Login(ctx context.Context, u dto.LoginDTO) (*response.UserAuthToken, error)
  }
```

o `Login` vai receber nosso contexto, dto e vai devolver um response que acabamos de criar.

Agora vamos implementar, dentro do arquivo recém criado `auth_service.go`:

```go
  func (s *service) Login(ctx context.Context, u dto.LoginDTO) (*response.UserAuthToken, error) {
    user, err := s.repo.FindUserByEmail(ctx, u.Email)
    if err != nil {
      slog.Error("error to search user by email", "err", err, slog.String("package", "userservice"))
      return nil, errors.New("error to search user password")
    }
    if user == nil {
      slog.Error("user not found", slog.String("package", "userservice"))
      return nil, errors.New("user not found")
    }

    return nil, nil
  }
```

Primeiro fazemos as validações básicas, verificando se o usuário com o e-mail informado realmente existe, nossa próxima validação requer criar um novo método no nosso `repository`, vai ser responsável por buscar apenas a senha do usuário, essa é uma boa prática, devemos evitar retornar a senha do usuário de forma desnecessária, por isso optei em criar um método único para isso.

Vamos criar primeiro a interface:

```go
  type UserRepository interface {
    CreateUser(ctx context.Context, u *entity.UserEntity) error
    FindUserByEmail(ctx context.Context, email string) (*entity.UserEntity, error)
    FindUserByID(ctx context.Context, id string) (*entity.UserEntity, error)
    UpdateUser(ctx context.Context, u *entity.UserEntity) error
    DeleteUser(ctx context.Context, id string) error
    FindManyUsers(ctx context.Context) ([]entity.UserEntity, error)
    UpdatePassword(ctx context.Context, pass, id string) error
    GetUserPassword(ctx context.Context, id string) (*entity.UserEntity, error)
  }
```

Agora vamos implementar:

```go
  func (r *repository) GetUserPassword(ctx context.Context, id string) (*entity.UserEntity, error) {
    userMock := entity.UserEntity{
      ID:       "1",
      Password: "$2y$12$CwjjXJGAkR4OKQeTvMo9suJ1s6PdKl9l4RZL9/yg.8cccDE8o/5sm",
    }
    return &userMock, nil
  }
```

Como nosso `repository` ainda não está pronto, vamos retornar dados fakes, sem isso não vamos conseguir validar e finalizar o service, por isso criar um hash da senha `12345678@`.

Vamos retornar um dado face no nosso método `FindUserByEmail` também:

```go
  func (r *repository) FindUserByEmail(ctx context.Context, email string) (*entity.UserEntity, error) {
    userMock := entity.UserEntity{
      ID:    "1",
      Name:  "John",
      Email: "john.doe@email.com",
    }
    return &userMock, nil
  }
```

Esses dados fakes são apenas para conseguir validar da forma correta o nosso service, na parte 6 vamos alterar.

Agora voltando ao service, podemos validar a senha, como o vamos comparar a senha? Uma vez criado o hash da senha se torna impossível reverter (até seria possível, mas não devemos ter poder computacional para tal feito atualmente), para poder comparar a senha vamos pegar a senha que o usuário informar no login, fazer novamente o hash e comparar com o hash que temos salvo no banco de dados, simples assim, o pacote do bcrypt do Go já permite comparar hashes de forma nativa, vamos ver:

```go
  userPass, err := s.repo.GetUserPassword(ctx, user.ID)
  if err != nil {
    slog.Error("error to search user password", "err", err, slog.String("package", "userservice"))
    return nil, errors.New("error to search user password")
  }
  // compare password with password in database
  err = bcrypt.CompareHashAndPassword([]byte(userPass.Password), []byte(u.Password))
  if err != nil {
    slog.Error("invalid password", slog.String("package", "userservice"))
    return nil, errors.New("invalid password")
  }
```

Primeiro buscamos a senha do usuário, depois com o `CompareHashAndPassword` fazemos a comparação entre `userPass.Password` (senha do banco) e `u.Password` (senha informada no login), se não retornar um erro então a senha está correta. Com a senha correta o próximo passo é gerar o token:

```go
  _, token, _ := env.Env.TokenAuth.Encode(map[string]interface{}{
    "id":    user.ID,
    "email": u.Email,
    "name":  user.Name,
    "exp":   time.Now().Add(time.Second * time.Duration(env.Env.JwtExpiresIn)).Unix(),
  })
```

Vamos chamar nosso `TokenAuth` como a função `Encode`, e vamos passar os dados que desejamos salver no token, na opção `exp` é onde definimos a expiração do token, com o valor informamos na nossa variável de ambiente.

Com isso temos o nosso token jwt pronto, basta retornar:

```go
  userAuthToken := response.UserAuthToken{
    AccessToken: token,
  }
  return &userAuthToken, nil
```

### Criando nosso handler

O handler vai ser muito semelhante ao que já fizemos nos outros posts. Vamos separar em outro arquivo assim como o service, para isso vamos criar um arquivo dentro do **handler/userhandler** chamado `auth_handler.go`:

Vamos primeiro criar a interface:

```go
  type UserHandler interface {
    CreateUser(w http.ResponseWriter, r *http.Request)
    UpdateUser(w http.ResponseWriter, r *http.Request)
    GetUserByID(w http.ResponseWriter, r *http.Request)
    DeleteUser(w http.ResponseWriter, r *http.Request)
    FindManyUsers(w http.ResponseWriter, r *http.Request)
    UpdateUserPassword(w http.ResponseWriter, r *http.Request)
    Login(w http.ResponseWriter, r *http.Request)
  }
```

Agora vamos implementar:

```go
  func (h *handler) Login(w http.ResponseWriter, r *http.Request) {
    if r.Body == http.NoBody {
      slog.Error("body is empty", slog.String("package", "userhandler"))
      w.WriteHeader(http.StatusBadRequest)
      msg := httperr.NewBadRequestError("body is required")
      json.NewEncoder(w).Encode(msg)
      return
    }
    var req dto.LoginDTO
    if r.Body != nil {
      err := json.NewDecoder(r.Body).Decode(&req)
      if err != nil {
        slog.Error("error to decode body", err, slog.String("package", "userhandler"))
        w.WriteHeader(http.StatusBadRequest)
        msg := httperr.NewBadRequestError("error to decode body")
        json.NewEncoder(w).Encode(msg)
        return
      }
    }
    httpErr := validation.ValidateHttpData(req)
    if httpErr != nil {
      slog.Error(fmt.Sprintf("error to validate data: %v", httpErr), slog.String("package", "userhandler"))
      w.WriteHeader(httpErr.Code)
      json.NewEncoder(w).Encode(httpErr)
      return
    }
    token, err := h.service.Login(r.Context(), req)
    if err != nil {
      if err.Error() == "user not found" || err.Error() == "invalid password" {
        w.WriteHeader(http.StatusUnauthorized)
        msg := httperr.NewUnauthorizedRequestError("invalid credentials")
        json.NewEncoder(w).Encode(msg)
        return
      }
      w.WriteHeader(http.StatusBadRequest)
      msg := httperr.NewBadRequestError(err.Error())
      json.NewEncoder(w).Encode(msg)
      return
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(token)
  }
```

Fazemos as mesmas validações das outras rotas, já vimos isso nos outros posts.

Vamos testar:

```http
POST http://localhost:8080/auth/login HTTP/1.1
content-type: application/json

{
  "email": "john.doe@email.com",
  "password": "12345678@"
}
```

Recebemos o retorno:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 04 Jan 2024 22:59:26 GMT
Content-Length: 195
Connection: close

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImpvaG4uZG9lQGVtYWlsLmNvbSIsImV4cCI6MTcwNDQyOTE2NSwiaWQiOiIxIiwibmFtZSI6IkpvaG4ifQ.lTFzOEh7RSVdEoM3WnWvlY4KeQq2G8cDMdluYovOQGs"
}
```

Sucesso! Temos nosso token válido, se quiser ver o conteúdo do token, pode usar este [site](https://jwt.io/), veja que contém as informações que colocamos no service:

![jwt io](fd55fgf4avjs&fh.png)

### Protegendo nossas rotas

Agora que temos o jwt precisamos proteger nossas rotas, as únicas rotas que podem ser chamadas sem a necessidade de informar um token serão a rota de login e criação de conta, para isso vamos alterar o arquivo responsável por gerenciar nossas rotas o `user_route.go`:

```go
  func InitUserRoutes(router chi.Router, h userhandler.UserHandler) {
    router.Post("/user", h.CreateUser)
    router.Route("/user", func(r chi.Router) {
      r.Use(jwtauth.Verifier(env.Env.TokenAuth))
      r.Use(jwtauth.Authenticator)

      r.Patch("/{id}", h.UpdateUser)
      r.Get("/{id}", h.GetUserByID)
      r.Delete("/{id}", h.DeleteUser)
      r.Get("/", h.FindManyUsers)
      r.Patch("/password/{id}", h.UpdateUserPassword)
    })
    router.Route("/auth", func(r chi.Router) {
      r.Post("/login", h.Login)
    })
  }
```

Adicionamos o `r.Use(jwtauth.Verifier(env.Env.TokenAuth))` do pacote go-chi que vai fazer a validação do token e verificar se é um token válido com o nosso segredo `JWT_SECRET` e se o token ainda está válido. O ` r.Use(jwtauth.Authenticator)` informa que é preciso informar um token jwt válido.

A nossa rota `router.Post("/user", h.CreateUser)` é uma rota aberta, não precisa informar um token, todas as rotas dentro grupo `Route` são rotas que necessitam de token.

Agora se testar nossas rotas autenticadas sem informar um token, vamos receber um status code `401`:

```http
GET http://localhost:8080/user HTTP/1.1
content-type: application/json
```

Resposta:

```http
GET http://localhost:8080/user HTTP/1.1
content-type: application/json
```

Adicionando um token:

```http
GET http://localhost:8080/user HTTP/1.1
content-type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImpvaG4uZG9lQGVtYWlsLmNvbSIsImV4cCI6MTcwNDQyOTE2NSwiaWQiOiIxIiwibmFtZSI6IkpvaG4ifQ.lTFzOEh7RSVdEoM3WnWvlY4KeQq2G8cDMdluYovOQGs
```

Resposta:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 04 Jan 2024 23:16:18 GMT
Content-Length: 15
Connection: close

{
  "users": []
}
```

Retorna um `user: []` pois ainda não implementamos o repository.

## Capturando o token

Vamos criar uma função que vai receber um token e retornar uma `struct` dos dados do token, isso vai facilitar o uso dos dados contidos no token.

Vamos criar uma pasta chamada **common** dentro da **internal** e dentro do **common** outra pasta chamada **utils** e um arquivo chamado `decode_jwt.go`, vai ficar assim `internal/common/utils/decode_jwt.go`:

```go
  type CurrentUser struct {
    ID        string `json:"id"`
    Email     string `json:"email"`
    Name      sstring `json:"name"`
    Exp       int64  `json:"exp,omitempty"`
    jwt.RegisteredClaims
  }
```

Primeiro vamos criar uma `struct` com os dados que vamos capturar no token, esses são os dados que informamos dentro do service quando geramos o token.

O `jwt.RegisteredClaims` é uma estrutura incorporada no pacote [github.com/golang-jwt/jwt/v4](github.com/golang-jwt/jwt/v4) que representa as reivindicações (claims) padrão definidas no nosso jwt, incluem campos como exp (tempo de expiração), iss (emissor), sub (assunto), entre outros.

Ao adicionar o `jwt.RegisteredClaims` na estrutura `CurrentUser`, estamos aproveitando esses dados. Isso permite que o pacote jwt interprete automaticamente e manipule essas reivindicações durante a validação e a análise do token jwt.

Vamos ler o token e transformar na `struct`:

```go
  func DecodeJwt(r *http.Request) (*CurrentUser, error) {
    authHeader := r.Header.Get("Authorization")
    parts := strings.Split(authHeader, " ")
    if len(parts) != 2 || parts[0] != "Bearer" {
      return nil, errors.New("invalid authorization header")
    }

    tokenString := parts[1]
    key := &env.Env.JwtSecret
    var userClaim CurrentUser

    _, err := jwt.ParseWithClaims(tokenString, &userClaim, func(token *jwt.Token) (interface{}, error) {
      return []byte(*key), nil
    })
    if err != nil {
      return nil, err
    }
    return &userClaim, nil
  }
```

Com isso temos nossa função pronta, mas onde usar? Vamos aplicar um exemplo, na rota que busca os dados do usuário pelo id, ao usar essa função não precisamos mais que seja informado um id, como essa rota é protegida por um token jwt e nosso token contém o id do usuário, podemos aproveitar dessa informação, vamos refatorar nosso `user_handler.go`.

```go
  func (h *handler) GetUserByID(w http.ResponseWriter, r *http.Request) {
    user, err := utils.DecodeJwt(r)
    if err != nil {
      slog.Error("error to decode jwt", slog.String("package", "userhandler"))
      w.WriteHeader(http.StatusBadRequest)
      msg := httperr.NewBadRequestError("error to decode jwt")
      json.NewEncoder(w).Encode(msg)
      return
    }
    res, err := h.service.GetUserByID(r.Context(), user.ID)
    if err != nil {
      slog.Error(fmt.Sprintf("error to get user: %v", err), slog.String("package", "userhandler"))
      if err.Error() == "user not found" {
        w.WriteHeader(http.StatusNotFound)
        msg := httperr.NewNotFoundError("user not found")
        json.NewEncoder(w).Encode(msg)
        return
      }
      w.WriteHeader(http.StatusInternalServerError)
      msg := httperr.NewBadRequestError("error to get user")
      json.NewEncoder(w).Encode(msg)
      return
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(res)
  }
```

Agora podemos chamar a função `DecodeJwt` e pegar o id do usuário, sem a necessidade do cliente informar o id. Podemos fazer isso para todos os enpoints que sejam autenticados e precise passar o id do usuário, no nosso caso são eles: `GetUserByID`, `DeleteUser`, `UpdateUserPassword` e `UpdateUser`.

Precisamos atualizar a documentação, removendo o `{id}` da url:

```go
  // User details
  //	@Summary		User details
  //	@Description	Get user by id
  //	@Tags			user
  //	@Security		ApiKeyAuth
  //	@Accept			json
  //	@Produce		json
  //	@Param			id	path		string	true	"user id"
  //	@Success		200	{object}	response.UserResponse
  //	@Failure		400	{object}	httperr.RestErr
  //	@Failure		404	{object}	httperr.RestErr
  //	@Failure		500	{object}	httperr.RestErr
  //	@Router			/user [get]
```

Rode o comando abaixo para atualizar o arquivo do swaggo:

```bash
  swag init -g internal/handler/routes/docs_route.go
```

## Criando um middleware

Por último, vamos criar um middleware que vai capturar os dados dos usuários que solicitarem a requisição a nossa api.

Vamos criar um pasta chamada **middleware** dentro de **handler** e um arquivo chamado `logger_middleware.go`:

```go
  func LoggerData(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      var requestData map[string]interface{}
      if r.Body != http.NoBody {
        // copy body
        CopyBody, _ := io.ReadAll(r.Body)
        // restore body
        r.Body = io.NopCloser(bytes.NewBuffer(CopyBody))
        if err := json.Unmarshal(CopyBody, &requestData); err != nil {
          slog.Error("error unmarshalling request data", err, slog.String("func", "LoggerData"))
        }
      } else {
        r.Body = http.NoBody
      }

      // get user in token
      var userID string
      var userEmail string
      user, err := utils.DecodeJwt(r)
      if err != nil {
        userID = "no token"
        userEmail = "no token"

      } else {
        userID = user.ID
        userEmail = user.Email
      }
      slog.Info("request_data",
        slog.Any("url", r.URL.Path),
        slog.Any("method", r.Method),
        slog.Any("query", r.URL.Query()),
        slog.Any("body", requestData),
        slog.Any("id", userID),
        slog.Any("email", userEmail),
      )

      next.ServeHTTP(w, r)
    })
  }
```

Esse middleware vai receber a requisição que foi chamada pelo cliente, lembrando que um middleware é iniciando logo após uma requisição ser iniciado, no nosso caso vai ser executado antes mesmo de chamar o nosso handler.

Vou explicar por etapas:

1 - Verificamos se o `body` não é vazio, com isso fazemos uma cópia para o `requestData`, como o body é um stream e só pode ser lido uma vez, sem a cópia nosso handler não conseguiria ler o body.
2 - Chamamos o nosso decode de jwt que criamos anteriormente `DecodeJwt`.
3 - Colocamos as informações que precisamos no log.

Agora nas rotas `user_route.go`, vamos usar o log de forma global:

```go
  func InitUserRoutes(router chi.Router, h userhandler.UserHandler) {
    router.Use(middleware.LoggerData)

    router.Post("/user", h.CreateUser)
    router.Route("/user", func(r chi.Router) {
      r.Use(jwtauth.Verifier(env.Env.TokenAuth))
      r.Use(jwtauth.Authenticator)

      r.Patch("/", h.UpdateUser)
      r.Get("/", h.GetUserByID)
      r.Delete("/", h.DeleteUser)
      r.Get("/list-all", h.FindManyUsers)
      r.Patch("/password", h.UpdateUserPassword)
    })
    router.Route("/auth", func(r chi.Router) {
      r.Post("/login", h.Login)
    })
  }
```

Adicionamos o `router.Use(middleware.LoggerData)`.

Precisamos remover da url o id `{id}` e na rota de listar todos os usuários adicione o `/list-all` para não gerar conflitos entre a rota de listar detalhes de um único usuário.

Veja como ficou, vamos fazer o login na nossa aplicacão:

```http
POST http://localhost:8080/auth/login HTTP/1.1
content-type: application/json

{
  "email": "john.doe@email.com",
  "password": "12345678@"
}
```

Nosso log:

```json
{
  "time": "2024-01-04T20:48:04.47413-03:00",
  "level": "INFO",
  "msg": "request_data",
  "url": "/auth/login",
  "method": "POST",
  "query": {},
  "body": { "email": "john.doe@email.com", "password": "12345678@" },
  "user_id": "no token",
  "user_email": "no token"
}
```

Repare que temos a senha exposta, precisamos remover dados sensíveis do log, para isso vamos criar uma função que retorna um booleano caso encontre a palavra que desejamos ocultar:

`logger_middleware.go`:

```go
  var sensitiveKeywords = []string{"password"}

  func hasSensitiveData(body map[string]interface{}) bool {
    for key := range body {
      for _, keyword := range sensitiveKeywords {
        if strings.Contains(strings.ToLower(key), keyword) || strings.Contains(strings.ToLower(body[key].(string)), keyword) {
          return true
        }
      }
    }
    return false
  }
```

Vamos ter um slice que oculta a palavra caso encontre no body:

```go
  func LoggerData(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      var requestData map[string]interface{}
      if r.Body != http.NoBody {
        // copy body
        CopyBody, _ := io.ReadAll(r.Body)
        // restore body
        r.Body = io.NopCloser(bytes.NewBuffer(CopyBody))
        if err := json.Unmarshal(CopyBody, &requestData); err != nil {
          slog.Error("error unmarshalling request data", err, slog.String("func", "LoggerData"))
        }
        if hasSensitiveData(requestData) {
          for key := range requestData {
            for _, keyword := range sensitiveKeywords {

              if strings.Contains(strings.ToLower(key), keyword) || strings.Contains(strings.ToLower(requestData[key].(string)), keyword) {
                requestData[key] = "[REDACTED]"
              }
            }
          }
        }
      } else {
        r.Body = http.NoBody
      }

      // get user in token
      var userID string
      var userEmail string
      user, err := utils.DecodeJwt(r)
      if err != nil {
        userID = "no token"
        userEmail = "no token"

      } else {
        userID = user.ID
        userEmail = user.Email
      }
      slog.Info("request_data",
        slog.Any("url", r.URL.Path),
        slog.Any("method", r.Method),
        slog.Any("query", r.URL.Query()),
        slog.Any("body", requestData),
        slog.Any("id", userID),
        slog.Any("email", userEmail),
      )

      next.ServeHTTP(w, r)
    })
  }
```

Caso encontre colocamos `"[REDACTED]"`, veja agora o log:

```json
{
  "time": "2024-01-04T21:11:44.850656-03:00",
  "level": "INFO",
  "msg": "request_data",
  "url": "/auth/login",
  "method": "POST",
  "query": {},
  "body": { "email": "john.doe@email.com", "password": "[REDACTED]" },
  "user_id": "no token",
  "user_email": "no token"
}
```

Vamos ver o log em uma rota autenticada:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 05 Jan 2024 00:12:22 GMT
Content-Length: 15
Connection: close

{
  "users": []
}
```

log:

```json
{
  "time": "2024-01-04T21:14:30.90976-03:00",
  "level": "INFO",
  "msg": "request_data",
  "url": "/user/list-all",
  "method": "GET",
  "query": {},
  "body": null,
  "user_id": "1",
  "user_email": "john.doe@email.com"
}
```

Esse middleware serve apenas para abordar sobre o uso de middlewares, seria necessário melhorar essas validações se forem utilizados logs dessa maneira. Para exibir dados de uma forma mais concisa, o log deveria estar dentro do handler, assim conseguimos controlar melhor os dados do body, uma vez que transformamos em struct e sabemos seu tipo, um middleware global fica um pouco complicado saber o valor do body, por isso criamos a função `hasSensitiveData` para auxiliar nessa tarefa.

## Considerações finais

Nesse post conseguimos finalizar a autenticação da nossa api, essa foi uma abordagem simples, existem outras estratégias mais avançadas que poderíamos utilizar como a utilização de refresh tokens, mas isso fica para um próximos posta

Se inscreva e receba um aviso quando sair novos posts, [se inscrever](https://wiliamvj.substack.com/)

## Próximos passos

Na parte 6 vamos iniciar nosso repository e persistir os dados no nosso banco de dados.

## Link do repositório

[repositório](https://github.com/wiliamvj/api-users-golang) do projeto

[Gopher credits](https://github.com/egonelbre/gophers)
