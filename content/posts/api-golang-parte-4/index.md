---
title: 'API completa em Golang - Parte 4'
date: 2023-12-21T17:54:40-03:00
draft: false
tableOfContents: false
thumbnail: 'htyf78HJsvdfczaqfg6jiopl.png'
---

![thumbnail](htyf78HJsvdfczaqfg6jiopl.png)

## O que vamos fazer?

Na parte 4 do nosso crud, vamos fazer a lógica do nosso service de usuário.

Se ainda não viu os posts anteriores leia eles primeiro.

[parte 1](posts/api-golang-parte-1/) |
[parte 2](posts/api-golang-parte-2/) |
[parte 3](/posts/api-golang-parte-3/) |

## Criando nossas entidades

Vamos primeiro criar nossa entidade de usuário, essa entidade vai ser uma representação dos dados do usuário no banco de dados, a entidade vai ser o que vamos retornar do nosso repository, dessa forma não vamos depender da tipagem geradao pelo sqlc. Precisamos criar nessa etapa para poder deixar o contrato entre `service` e `repository` definidos.

Crie um arquivo na pasta **entity** chamado `user_entity.go`:

```go
  type UserEntity struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Password  string    `json:"password,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
  }
```

Por hora essa serão os dados da entidade user.

## Contrato entre service e repository

Ainda não vamos trabalhar no service, mas precisamos definir o contrato para deixar nosso service caminhando sem o repository estar pronto, os contratos nos ajudam com isso, esse desacoplamento permite mais liberdade de avançar sem precisar implementar outras partes da aplicação, podemos inclusive adiar a escolha do banco de dados a ser utilizado, mas poderíamos tranquilamente finalizar todo o resto da aplicação.

Vamos criar nossas interfaces no `user_interface_repository.go`:

```go
  func NewUserRepository(db *sql.DB, q *sqlc.Queries) UserRepository {
    return &repository{
      db,
      q,
    }
  }

  type repository struct {
    db      *sql.DB
    queries *sqlc.Queries
  }

  type UserRepository interface {
    CreateUser(ctx context.Context, u *entity.UserEntity) error
    FindUserByEmail(ctx context.Context, email string) (*entity.UserEntity, error)
    FindUserByID(ctx context.Context, id string) (*entity.UserEntity, error)
    UpdateUser(ctx context.Context, u *entity.UserEntity) error
    DeleteUser(ctx context.Context, id string) error
    FindManyUsers(ctx context.Context) ([]entity.UserEntity, error)
    UpdatePassword(ctx context.Context, pass, id string) error
  }
```

Por enquanto esses serão o métodos que vamos utilizar para o usuário, sempre que precisamos retornamos nossa entidade, no qual é de nosso controle, como já mencionado, poderíamos remover essa abstração e o service chamar diretamente os métodos e tipagem gerados pelo sqlc, mas ficamos amarrados ao sqlc, dessa forma ficaria mais tranquilo remover o sqlc caso fosse necessário.

Implementando o `UserRepository`:

```go
  func (r *repository) CreateUser(ctx context.Context, u *entity.UserEntity) error {
    return nil
  }

  func (r *repository) FindUserByEmail(ctx context.Context, email string) (*entity.UserEntity, error) {
    return nil, nil
  }

  func (r *repository) FindUserByID(ctx context.Context, id string) (*entity.UserEntity, error) {
    return nil, nil
  }

  func (r *repository) UpdateUser(ctx context.Context, u *entity.UserEntity) error {
    return nil
  }

  func (r *repository) DeleteUser(ctx context.Context, id string) error {
    return nil
  }

  func (r *repository) FindManyUsers(ctx context.Context) ([]entity.UserEntity, error) {
    return nil, nil
  }

  func (r *repository) UpdatePassword(ctx context.Context, pass, id string) error {
    return nil
  }
```

Como não vamos trabalhar nessa camada agora, apenas implemente o contrato para o Go parar de acusar o erro.

## Implementando o service

Agora vamos partir para a nossa regra de negócio, o service é a camada mais importante, nela que vamos definir o comportamento da nossa aplicação, no handler apenas tratamos a entrada do dado, não manipulamos nada, e o repository vai servir apenas para persistir e buscar dados no banco.

### Criando o usuário

Primeira coisa que precisamos fazer é verificar se o e-mail do usuário que vamos cadastrar já existe no banco, pois nosso campo de e-mail no banco será único

```go
  userExists, err := s.repo.FindUserByEmail(ctx, u.Email)
  if err != nil {
    slog.Error("error to search user by email", "err", err, slog.String("package", "userservice"))
    return err
  }
  if userExists != nil {
    slog.Error("user already exists", slog.String("package", "userservice"))
    return errors.New("user already exists")
  }
```

Se a busca do usuário por e-mail retornar um erro, retornamos esse erro, se o `userExists` for diferente de nil, significa que encontrou um usuário, então não podemos deixar prosseguir, retornamos um novo erro `errors.New("user already exists")`, ainda vamos tratar esse erro no handler e retornar um status code correto.

```go
  passwordEncrypted, err := bcrypt.GenerateFromPassword([]byte(u.Password), 12)
  if err != nil {
    slog.Error("error to encrypt password", "err", err, slog.String("package", "userservice"))
    return errors.New("error to encrypt password")
  }
```

Agora vamos precisar encriptar a senha do usuário, usando o pacote nativo do Go chamado [bcrypt](https://pkg.go.dev/golang.org/x/crypto@v0.13.0/bcrypt), transformamos a senha do usuário que está no `u.Password` em um slice de bytes e informamos que a força da encriptação da senha é 12, por padrão o `DefaultCost` é 10, quanto maior esse número mais forte é sua encriptação, o máximo aceito é 31, lembrando que encriptar tem um custo computacional, quanto maior esse número mais tempo vai levar.

Por curiosidade usando o valor 12 demora cerca de 100ms (Milissegundo) para encriptar, colocando 31 demora cerca de 7 minutos, claro que depende do hardware que está fazendo isso, mas a diferença de tempo é muito grande. Por padrão o uso comum usamos entre 10 e 14.

Agora vamos transformar os dados recebido para a nossa estrutura do `UserEntity`:

```go
  newUser := entity.UserEntity{
    ID:        uuid.New().String(),
    Name:      u.Name,
    Email:     u.Email,
    Password:  string(passwordEncrypted),
    CreatedAt: time.Now(),
    UpdatedAt: time.Now(),
  }
```

Para criar o id do tipo uuid usamos o [pacote da google](https://pkg.go.dev/github.com/google/uuid@v1.1.2) o mesmo que usamos para validar um uuid no handler. Outra vantagem de remover a responsabilidade da criação do id do banco de dados, é que agora já sabemos o id do desse usuário sem salvar no banco, isso tem inúmeras vantagem.

Por fim vamos chamar nosso repository e salvar o usuário no banco:

```go
  err = s.repo.CreateUser(ctx, &newUser)
  if err != nil {
    slog.Error("error to create user", "err", err, slog.String("package", "userservice"))
    return err
  }
```

Apenas isso é suficiente para criar um usuário, ainda vamos adicionar o endereço do usuário ainda na parte 4, e vamos alterar o `CreateUser`.

### Atualizando o usuário

Vai ser parecido com a criação, mas primeiro vamos verificar se o usuário existe pelo id:

```go
  userExists, err := s.repo.FindUserByID(ctx, id)
  if err != nil {
    slog.Error("error to search user by id", "err", err, slog.String("package", "userservice"))
    return err
  }
  if userExists == nil {
    slog.Error("user not found", slog.String("package", "userservice"))
    return errors.New("user already exists")
  }
```

A lógica inverte, na criação se o `userExists` for diferente de `nil` retornamos um erro, no update se o `userExists` for igual a `nil`, retornamos um erro, pois significa que não encontramos o usuário que o client que atualizar.

Tem um detalhe, caso o e-mail seja informado:

```go
  if u.Email != "" {
    verifyUserEmail, err := s.repo.FindUserByEmail(ctx, u.Email)
    if err != nil {
      slog.Error("error to search user by email", "err", err, slog.String("package", "userservice"))
      return err
    }
    if verifyUserEmail != nil {
      slog.Error("user already exists", slog.String("package", "userservice"))
      return errors.New("user already exists")
    }
  }
```

Caso o client precise atualizar o e-mail, antes precisamos verificar se o novo e-mail não está sendo utilizado por outro usuário, fazemos isso no código acima.

Por fim, vamos passar tudo isso para a struct `UserEntity` e chamar nosso repository:

```go
  updateUser := entity.UserEntity{
    ID:        id,
    Name:      u.Name,
    Email:     u.Email,
    UpdatedAt: time.Now(),
  }
  err = s.repo.UpdateUser(ctx, &updateUser)
  if err != nil {
    slog.Error("error to update user", "err", err, slog.String("package", "userservice"))
    return err
  }
  return nil
```

E retornamos `nil` se todo ocorrer corretamente, perceba que na criação, atualização, deletar usuário retornamos sempre `nil`, na nossa aplicação não tem sentido retornar dados para o cliente, pois são os mesmos dados que acabamos de receber do client, mas existem casos que faz sentido retornar após nossa api processar os dados, tudo depende da sua regra.

### Detalhes do usuário

Vamos buscar os dados do usuário pelo id.

```go
 userExists, err := s.repo.FindUserByID(ctx, id)
  if err != nil {
    slog.Error("error to search user by id", "err", err, slog.String("package", "userservice"))
    return nil, err
  }
  if userExists == nil {
    slog.Error("user not found", slog.String("package", "userservice"))
    return nil, errors.New("user not found")
  }
```

Mesma lógica que já fizemos acima, porém agora nosso service `GetUserByID` retorna um `*response.UserResponse`, então precisamos passar nosso `UserEntity` retornado pelo repository para `UserResponse`:

```go
  user := response.UserResponse{
    ID:        userExists.ID,
    Name:      userExists.Name,
    Email:     userExists.Email,
    CreatedAt: userExists.CreatedAt,
    UpdatedAt: userExists.UpdatedAt,
  }
  return &user, nil
```

E pronto, isso é o suficiente!

### Deletando o usuário

Essa deve ser a mais simples de todas:

```go
  userExists, err := s.repo.FindUserByID(ctx, id)
  if err != nil {
    slog.Error("error to search user by id", "err", err, slog.String("package", "userservice"))
    return err
  }
  if userExists == nil {
    slog.Error("user not found", slog.String("package", "userservice"))
    return errors.New("user not found")
  }
  err = s.repo.DeleteUser(ctx, id)
  if err != nil {
    slog.Error("error to delete user", "err", err, slog.String("package", "userservice"))
    return err
  }
  return nil
```

Apenas verificamos se o usuário existe e depois chamamos o repository.

### Listar todos os usuários

Esse metódo também é bem simples.

```go
  findManyUsers, err := s.repo.FindManyUsers(ctx)
  if err != nil {
    slog.Error("error to find many users", "err", err, slog.String("package", "userservice"))
    return nil, err
  }
  users := response.ManyUsersResponse{}
  for _, user := range findManyUsers {
    userResponse := response.UserResponse{
      ID:        user.ID,
      Name:      user.Name,
      Email:     user.Email,
      CreatedAt: user.CreatedAt,
      UpdatedAt: user.UpdatedAt,
    }
    users.Users = append(users.Users, userResponse)
  }
  return &users, nil
```

Como nosso repository retorna um slice do `[]entity.UserEntity`, precisamos fazer um loop e transformar em `response.ManyUsersResponse`, é o que fazemos acima.

### Alterando a senha

Por último, vamos atualizar a senha.

Primeiro fazemos a validação padrão se o usuário existe.

```go
  userExists, err := s.repo.FindUserByID(ctx, id)
  if err != nil {
    slog.Error("error to search user by id", "err", err, slog.String("package", "userservice"))
    return err
  }
  if userExists == nil {
    slog.Error("user not found", slog.String("package", "userservice"))
    return errors.New("user not found")
  }
```

Agora precisamos verificar se a senha que temos no banco de dados é a mesma senha que o usuário informou como sua senha antiga, que usamos no nosso dto `UpdateUserPasswordDto`, se for diferente já retornamos um erro:

```go
  // compare passwords
  err = bcrypt.CompareHashAndPassword([]byte(userExists.Password), []byte(u.OldPassword))
  if err != nil {
    slog.Error("invalid password", slog.String("package", "userservice"))
    return errors.New("invalid password")
  }
```

Agora vamos adicionar mais um verificação de segurança, para impedir que o usuário troque uma senha por outra exatamente igua, mesma lógica acima, mas agora comparando com a senha do banco e a nova senha que o usuário informou:

```go
  // compare new password with password in database
  err = bcrypt.CompareHashAndPassword([]byte(userExists.Password), []byte(u.Password))
  if err == nil {
    slog.Error("new password is equal to old password", slog.String("package", "userservice"))
    return errors.New("new password is equal to old password")
  }
```

Por fim, vamos encriptar a nova senha do usuário e salvar no banco, igual o que fizemos na criação do usuário, porém criei um método apenas para salvar a senha do usuário no banco:

```go
  passwordEncrypted, err := bcrypt.GenerateFromPassword([]byte(u.Password), 12)
  if err != nil {
    slog.Error("error to encrypt password", "err", err, slog.String("package", "userservice"))
    return errors.New("error to encrypt password")
  }
  err = s.repo.UpdatePassword(ctx, string(passwordEncrypted), id)
  if err != nil {
    slog.Error("error to update password", "err", err, slog.String("package", "userservice"))
    return err
  }
  return nil
```

## Melhorando erros do handler

Nosso service retorna alguns erros que precisamos melhorar no nosso handler, como `"user not found"`, para isso vamos adicionar esse tratamento no `user_handler.go`:

```go
  err = h.service.UpdateUser(r.Context(), req, id)
  if err != nil {
    slog.Error(fmt.Sprintf("error to update user: %v", err), slog.String("package", "handler_user"))
    if err.Error() == "user not found" {
      w.WriteHeader(http.StatusNotFound)
      msg := httperr.NewNotFoundError("user not found")
      json.NewEncoder(w).Encode(msg)
      return
    }
    w.WriteHeader(http.StatusBadRequest)
    msg := httperr.NewBadRequestError("error to update user")
    json.NewEncoder(w).Encode(msg)
    return
  }
```

Dentro do tratamento do erro da chamada ao service, vamos adicionar um novo `if`, caso o erro seja do tipo `"user not found"` retornamos um erro com no nosso `httperr` do tipo `NotFoundError`.

Você pode alterar essa forma de tratar os erros, para evitar ficar tratando isso no handler, como por exemplo no service retornar já um erro do tipo `*http.Request`, assim no handler poderia apenas retornar o erro e o service que seria responsável por determinar o tipo do erro, status code e tudo mais, porém prefiro separar e deixar no handler a responsabilidade do http.

Você vai precisar colocar esse tratamento nos métodos do handler: `UpdateUser`, `GetUserByID`, `DeleteUser`, `UpdateUserPassword`.

## Salvando o endereço

Agora vamos salvar o endereço do usuário, para isso vamos utilizar a api do [viacep](https://viacep.com.br/), vamos primeiro atualizar nosso dto `CreateUserDto` e `UpdateUserDto`:

```go
  type CreateUserDto struct {
    Name     string `json:"name" validate:"required,min=3,max=30"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8,max=30,containsany=!@#$%*"`
    CEP      string `json:"cep" validate:"required,min=8,max=8"`
  }

  type UpdateUserDto struct {
    Name  string `json:"name" validate:"omitempty,min=3,max=30"`
    Email string `json:"email" validate:"omitempty,email"`
    CEP   string `json:"cep" validate:"omitempty,min=8,max=8"`
  }
```

Adicionando o campo `CEP`, com a validação simples de no mínimo e máximo 8 caracteres, como o nosso validator `ValidateHttpData` já tem o case `min` e `max` não precisamos fazer nada, já está funcionando a validação.

Vamos separar a chamada a api do viacep em uma função, vamos criar na raiz do projeto uma pasta chamada **api** e dentro dela outra pasta chamada **viacep**, na pasta api vamos salvar todo o código que for chamar api de terceiros, agora crie um arquivo chamado `viacep.go` dentro da pasta **api/viacep**:

```go
  package viacep

  import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/wiliamvj/api-users-golang/config/env"
  )

  type ViaCepResponse struct {
    CEP         string `json:"cep"`
    Logradouro  string `json:"logradouro"`
    Complemento string `json:"complemento"`
    Bairro      string `json:"bairro"`
    Localidade  string `json:"localidade"`
    UF          string `json:"uf"`
    IBGE        string `json:"ibge"`
    GIA         string `json:"gia"`
    DDD         string `json:"ddd"`
    SIAFI       string `json:"siafi"`
  }

  func GetCep(cep string) (*ViaCepResponse, error) {
    url := fmt.Sprintf("%s/%s/json", env.Env.ViaCepURL, cep)
    var viaCepResponse ViaCepResponse

    resp, err := http.Get(url)
    if err != nil {
      return nil, err
    }
    defer resp.Body.Close()

    err = json.NewDecoder(resp.Body).Decode(&viaCepResponse)
    if err != nil {
      return nil, err
    }
    if viaCepResponse.CEP == "" {
      return nil, fmt.Errorf("cep not found")
    }
    return &viaCepResponse, nil
  }
```

Primeiro criamos um struct para deixar com a tipagem da resposta do viacep, depois criamos uma função chamada `GetCep` que recebe uma cep do tipo string e retorna um ponteiro do `*ViaCepResponse` ou um `error`, depois montamos a url usando o `fmt.Sprintf`, o `ViaCepURL` colocamos em uma váriavel, você pode colocar a url diretamente se preferir, coloquei em uma env para treinar como utilizar a env com viper, precisa colocar o valor na env:

```go
  VIA_CEP_URL="https://viacep.com.br/ws"
```

Precisamos importar usando viper na nossa **config**:

```go
  type config struct {
    GoEnv       string `mapstructure:"GO_ENV"`
    GoPort      string `mapstructure:"GO_PORT"`
    DatabaseURL string `mapstructure:"DATABASE_URL"`
    ViaCepURL   string `mapstructure:"VIA_CEP_URL"`
  }
```

Depois fazemos a chamada [http](https://pkg.go.dev/net/http) usando o pacote do Go, por fim transformamos o body recebido na nossa struct `ViaCepResponse`.

Precisamos agora alterar nosso entity, para adicionar o endereço:

```go
  type UserEntity struct {
    ID        string      `json:"id"`
    Name      string      `json:"name"`
    Email     string      `json:"email"`
    Password  string      `json:"password,omitempty"`
    Address   UserAddress `json:"address,omitempty"`
    CreatedAt time.Time   `json:"created_at"`
    UpdatedAt time.Time   `json:"updated_at"`
  }

  type UserAddress struct {
    CEP        string `json:"cep"`
    IBGE       string `json:"ibge"`
    UF         string `json:"uf"`
    City       string `json:"city"`
    Complement string `json:"complement,omitempty"`
    Street     string `json:"street"`
  }
```

Não precisamos alterar nada no nosso handler, somente no nosso service, vamos lá! Vamos alterar somente o `CreateUser` e `UpdateUser`.

`CreateUser`:

```go
  cep, err := viacep.GetCep(u.CEP)
  if err != nil {
    slog.Error("error to get cep", "err", err, slog.String("package", "userservice"))
    return err
  }
  newUser := entity.UserEntity{
    ID:       uuid.New().String(),
    Name:     u.Name,
    Email:    u.Email,
    Password: string(passwordEncrypted),
    Address: entity.UserAddress{
      CEP:        cep.CEP,
      IBGE:       cep.IBGE,
      UF:         cep.UF,
      City:       cep.Localidade,
      Complement: cep.Complemento,
      Street:     cep.Logradouro,
    },
    CreatedAt: time.Now(),
    UpdatedAt: time.Now(),
  }
```

Primeiro vamos pegar o cep usando a função que criamos, caso retorne um erro paramos ali, depois apenas colocamos o retorno do `cep` na struct `newUser` e pronto, isso é suficiente.

O `UpdateUser` que vai ter um pouco mais de alteração, primeiro não vamos mais inicializar e já atribuir valor do `UserEntity` diretamente igual estávamos fazendo, vamos primeiro declarar:

```go
  var updateUser entity.UserEntity
  if u.Email != "" {
    verifyUserEmail, err := s.repo.FindUserByEmail(ctx, u.Email)
    if err != nil {
      slog.Error("error to search user by email", "err", err, slog.String("package", "userservice"))
      return err
    }
    if verifyUserEmail != nil {
      slog.Error("user already exists", slog.String("package", "userservice"))
      return errors.New("user already exists")
    }
    updateUser.Email = u.Email
  }
  if u.CEP != "" {
    cep, err := viacep.GetCep(u.CEP)
    if err != nil {
      slog.Error("error to get cep", "err", err, slog.String("package", "userservice"))
      return err
    }
    updateUser.Address = entity.UserAddress{
      CEP:        cep.CEP,
      IBGE:       cep.IBGE,
      UF:         cep.UF,
      City:       cep.Localidade,
      Complement: cep.Complemento,
      Street:     cep.Logradouro,
    }
  }
```

Primeiro declaramos o `updateUser`, depois se tiver um e-mail dentro do if atribuímos o `updateUser.Email = u.Email`, depois ao atualizar se houver um cep, entramos no if e vamos chamar a nossa função `GetCep` e por fim inicializamos e atribuímos o `updateUser.Address`.

Depois atribuímos os valores finais:

```go
  updateUser.ID = id
  updateUser.Name = u.Name
  updateUser.UpdatedAt = time.Now()
```

Existe várias maneiras de fazer isso, essa é a forma que acho mais simples de entender o que está acontecendo e o que estamos atribuindo a nosso `UserEntity`.

## Melhorando erros do handler

Podemos adicionar novos erros no nosso handler:

```go
err = h.service.CreateUser(r.Context(), req)
  if err != nil {
    slog.Error(fmt.Sprintf("error to create user: %v", err), slog.String("package", "userhandler"))
    if err.Error() == "cep not found" {
      w.WriteHeader(http.StatusNotFound)
      msg := httperr.NewNotFoundError("cep not found")
      json.NewEncoder(w).Encode(msg)
      return
    }
    w.WriteHeader(http.StatusInternalServerError)
    msg := httperr.NewBadRequestError("error to create user")
    json.NewEncoder(w).Encode(msg)
    return
  }
```

Adicionamos o erro `"cep not found"` para novos erro do cep, adicione apenas no `CreateUser` e `UpdateUser`.

## Considerações finais

Nesse post conseguimos avançar bastante a nossa api, deixamos nosso service de usuário praticamente pronto, fizemos o cadastro de endereço chamando uma api de terceiros, deixamos o contrato entre service e repository pronto, aos poucos estamos deixando nossa api mais completa e robusta.

Agora temos uma newsletter, se inscreva e receba um aviso quando sair novos posts, [se inscrever](https://wiliamvj.substack.com/)

## Próximos passos

Na parte 5 vamos partir para a autenticação do usuário, protegendo as rotas da nossa aplicação, criando um service separado para lidar com a criação do token JWT, vamos adicionar um middleware personalizado para capturar log de cada chamada http pegando dados do usuário.

## Link do repositório

[repositório](https://github.com/wiliamvj/api-users-golang) do projeto

[Gopher credits](https://github.com/egonelbre/gophers)
