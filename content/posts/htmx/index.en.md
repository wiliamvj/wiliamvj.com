---
title: "Creating Web Applications with HTMX and Golang"
date: 2026-07-24T17:00:30-03:00
draft: false
tableOfContents: false
thumbnail: "thumb_d873hdfdf23t2.png"
---

![thumbnail](thumb_d873hdfdf23t2.png)

Let's build a simple TODO list application using Golang, HTMX, SQLite and Tailwind CSS. The goal is to show how to develop a modern web application without relying on complex JavaScript frameworks, taking advantage of server-side rendering and partial page updates.

During development we will see how to structure the project in Go, create the application routes, persist data with SQLite and use HTMX to update the interface dynamically, without reloading the page. For the visuals, we will use Tailwind CSS.

## Structure of our example project

![Project structure](hgsdfjhgsdjw5435.png)

- **db**: Responsible for the connection with the SQLite database and database initialization.
- **handlers**: Contains the application's HTTP handlers, responsible for processing requests and rendering HTML templates.
- **models**: Defines the structures (models) used by the application, such as the `Todo` entity.
- **static/css**: Static files used by the application, such as fonts and custom styles.
- **templates**: Contains all HTML templates of the application.
  - **partials**: Reusable partial templates, such as the task display component.
- **main.go**: Application entry point, responsible for configuring dependencies, registering routes and starting the HTTP server.

## What is HTMX?

**HTMX** is a JavaScript library that allows adding interactivity to web applications in a declarative way, directly in the HTML. Instead of writing complex JavaScript code to make AJAX requests and update parts of the page, you use simple HTML attributes.

The central idea of HTMX is to extend HTML to allow any element on the page to make HTTP requests and update specific parts of the DOM with the server's response. All this without reloading the entire page.

This fits perfectly with Go applications that render templates on the server, since HTMX works with partial HTML responses — exactly what Go templates already do natively.

### Why use HTMX with Go?

Using HTMX in a Go application brings several advantages:

**1 - Simplicity**: No need to configure a complete REST API with JSON. The server responds directly with HTML and HTMX injects that HTML in the correct place on the page.
**2 - Less JavaScript**: The interactivity logic stays in the HTML, not in `.js` or `.ts` files. The Go server continues to be the place for the application's business rules.
**3 - Server-Side Rendering**: Takes full advantage of Go's native templates (`html/template`) to render partial components that HTMX dynamically replaces.
**4 - Progressive**: You can add HTMX to specific parts of the application without rewriting everything. A traditional page can gradually gain interactivity.
**5 - Lightness**: Lightweight library, with no dependencies.

## Setting Up the Project

Let's start with the base structure. The `main.go` configures the server, loads the templates and registers the routes:

```go
package main

import (
    "fmt"
    "html/template"
    "log"
    "net/http"
    "os"
    "path/filepath"
    "strings"

    "htmx/db"
    "htmx/handlers"
)

func main() {
    database := db.InitDB()
    defer database.Close()

    tmpl, err := parseTemplates()
    if err != nil {
        log.Fatalf("Erro ao carregar templates: %v", err)
    }

    todoHandler := handlers.NewTodoHandler(database, tmpl)

    mux := http.NewServeMux()

    fs := http.FileServer(http.Dir("static"))
    mux.Handle("/static/", http.StripPrefix("/static/", fs))

    mux.HandleFunc("/dashboard", todoHandler.Index)
    mux.HandleFunc("/todos/create", todoHandler.Create)
    mux.HandleFunc("/todos/toggle", todoHandler.Toggle)
    mux.HandleFunc("/todos/delete", todoHandler.Delete)
    mux.HandleFunc("/", todoHandler.Index)

    fmt.Println("Servidor rodando em http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}

func parseTemplates() (*template.Template, error) {
    funcMap := template.FuncMap{
        "add": func(a, b int) int { return a + b },
        "sub": func(a, b int) int { return a + b },
    }

    tmpl := template.New("").Funcs(funcMap)

    err := filepath.Walk("templates", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !info.IsDir() && strings.HasSuffix(path, ".html") {
            b, err := os.ReadFile(path)
            if err != nil {
                return err
            }

            name := filepath.Base(path)
            _, err = tmpl.New(name).Parse(string(b))
            if err != nil {
                return err
            }
        }
        return nil
    })

    return tmpl, err
}
```

The `parseTemplates` function loads all `.html` files from the templates directory and registers helper functions in the `FuncMap`, in this case, add and sub, which we will use later to calculate task statistics.

## The Application Templates

The application uses a layered template structure. The `layout.html` is the skeleton of the page and includes HTMX via CDN:

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>TODO</title>
    <link rel="stylesheet" href="/static/css/app.css" />
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/htmx.org@2.0.0"></script>
  </head>
  <body
    class="bg-slate-50 min-h-screen flex items-center justify-center p-4 antialiased text-slate-900"
  >
    <div
      class="w-full max-w-md bg-white rounded-xl border border-slate-200 shadow-sm overflow-hidden"
    >
      {{ template "index.html" . }}
    </div>
  </body>
</html>
```

Note that HTMX is loaded simply with a `<script>` tag. There is no build, no bundler, no extra configuration.

The `index.html` template contains the creation form and the task list:

```html
<div class="p-6 space-y-6">
  <form
    id="todo-form"
    hx-post="/todos/create"
    hx-target="#todo-list"
    hx-swap="innerHTML scroll:no-change"
    hx-on:htmx:after-request="document.getElementById('todo-form').reset()"
    class="flex gap-2"
  >
    <input
      type="text"
      name="title"
      placeholder="Add new task..."
      required
      class="flex h-10 w-full rounded-md border border-slate-200 bg-white px-3 py-2 text-sm ring-offset-white placeholder:text-slate-500 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-slate-950 focus-visible:ring-offset-2"
    />

    <button
      type="submit"
      class="inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-slate-950 focus-visible:ring-offset-2 bg-slate-900 text-slate-50 hover:bg-slate-900/90 h-10 px-4 py-2 shrink-0"
    >
      Add
    </button>
  </form>

  <div id="todo-list" class="max-h-[360px] overflow-y-auto pr-1">
    {{ template "todo-list.html" . }}
  </div>
</div>
```

Here come the HTMX attributes. Let's understand each one:

- `hx-post`: When the form is submitted, HTMX intercepts the event and makes a `POST` request to `/todos/create`, instead of reloading the page.

- `hx-target"`: Defines which element will be updated with the server's response. In this case, the `div` with id `todo-list`.

- `hx-swap=`: Specifies how the content will be inserted. `innerHTML` replaces the internal content of the target.

- `hx-on:htmx:after-request"`: Listens for the `htmx:after-request` event (triggered after the request finishes) and executes JavaScript to clear the form, leaving the input ready for the next task.

See more HTMX tags [here](https://htmx.org/reference/).

The `todo-list.html` template renders the list and the statistics:

```html
<div
  class="sticky top-0 bg-white z-10 flex items-center justify-between text-xs text-slate-500 pb-2 px-1 border-b border-slate-100 mb-3"
>
  {{ $total := len . }} {{ $completed := 0 }} {{ range . }} {{ if .Completed }}
  {{ $completed = add $completed 1 }} {{ end }} {{ end }}

  <div class="flex gap-2">
    <span
      class="inline-flex items-center rounded-full bg-slate-100 px-2.5 py-0.5 text-xs font-medium text-slate-700 border border-slate-200"
    >
      Pendentes:
      <strong class="ml-1 text-slate-900">{{ sub $total $completed }}</strong>
    </span>
    <span
      class="inline-flex items-center rounded-full bg-emerald-50 px-2.5 py-0.5 text-xs font-medium text-emerald-700 border border-emerald-200/60"
    >
      Concluídas:
      <strong class="ml-1 text-emerald-900">{{ $completed }}</strong>
    </span>
  </div>

  <span class="font-medium text-slate-400">Total: {{ $total }}</span>
</div>

<ul class="space-y-2">
  {{ range . }} {{ template "todo-item.html" . }} {{ else }}
  <p
    class="text-sm text-slate-500 text-center py-8 border border-dashed border-slate-200 rounded-lg bg-slate-50/50"
  >
    Nenhuma tarefa encontrada.
  </p>
  {{ end }}
</ul>
```

And each list item is a reusable partial template, `todo-item.html`:

```html
<li
  id="todo-{{ .ID }}"
  class="flex items-center justify-between p-3 rounded-lg border border-slate-100 bg-slate-50/50 hover:bg-slate-100/50 transition-colors"
>
  <div class="flex items-center space-x-3">
    <!-- prettier-ignore -->
    <input
      type="checkbox"
      {{ if .Completed }} checked {{ end }}
      hx-post="/todos/toggle?id={{ .ID }}"
      hx-target="#todo-list"
      hx-swap="innerHTML scroll:no-change"
      class="h-4 w-4 rounded border-slate-300 text-slate-900 focus:ring-slate-950 cursor-pointer"
    />
    <span
      class="text-sm font-medium {{ if .Completed }}line-through text-slate-400{{ else }}text-slate-700{{ end }}"
    >
      {{ .Title }}
    </span>
  </div>

  <button
    hx-delete="/todos/delete?id={{ .ID }}"
    hx-target="#todo-list"
    hx-swap="innerHTML scroll:no-change"
    class="inline-flex items-center justify-center rounded-md text-sm font-medium text-slate-400 hover:text-red-600 hover:bg-red-50 h-8 w-8 transition-colors"
  >
    <svg
      xmlns="http://www.w3.org/2000/svg"
      width="16"
      height="16"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="2"
      stroke-linecap="round"
      stroke-linejoin="round"
    >
      <path d="M3 6h18"></path>
      <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path>
      <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path>
    </svg>
  </button>
</li>
```

In the task item, we have two more HTMX attributes in action:

- `hx-post="/todos/toggle?id={{ .ID }}"`: on the checkbox: when checking or unchecking, it makes a `POST` request to toggle the task status and updates the list.

- `hx-delete="/todos/delete?id={{ .ID }}"`: on the delete button: makes a `DELETE` request to remove the task. HTMX supports standard HTTP methods directly in the attributes.

In both cases, `hx-target="#todo-list"` and `hx-swap="innerHTML scroll:no-change"` ensure that only the list is re-rendered, keeping the rest of the page intact.

## The Handlers in Go

The handlers process the requests and respond with partial HTML. See the creation handler:

```go
func (h *TodoHandler) Create(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Método não permitido", http.StatusMethodNotAllowed)
		return
	}

	title := r.FormValue("title")
	if strings.TrimSpace(title) == "" {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	_, err := h.db.Exec("INSERT INTO todos (title, completed) VALUES (?, ?)", title, false)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	todos, err := h.getAllTodos()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	err = h.tmpl.ExecuteTemplate(w, "todo-list.html", todos)
	if err != nil {
		log.Printf("Erro ao renderizar template todo-list.html: %v", err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

The important point here is the last line: `h.tmpl.ExecuteTemplate(w, "todo-list.html", todos)`. Instead of returning JSON, the handler renders the `todo-list.html` template directly in the HTTP response. HTMX receives this HTML and injects it into the `#todo-list `of the page.The same pattern repeats in the `toggle` and `delete` handlers:

```go
func (h *TodoHandler) Toggle(w http.ResponseWriter, r *http.Request) {
	// ... validação e update no banco ...

	todos, err := h.getAllTodos()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	err = h.tmpl.ExecuteTemplate(w, "todo-list.html", todos)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}

func (h *TodoHandler) Delete(w http.ResponseWriter, r *http.Request) {
	// ... validação e delete no banco ...

	todos, err := h.getAllTodos()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	err = h.tmpl.ExecuteTemplate(w, "todo-list.html", todos)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

All the application state logic stays on the server. HTMX only orchestrates the requests and DOM updates.

**How Does the Flow Work?**

1 - Initial Load: The user accesses `/dashboard`. The Index handler renders the complete `layout.html`, which includes index.html and the initial task list.

2 - Add Task: The user types in the form and clicks "Add". HTMX intercepts the submit, makes a `POST` to `/todos/create` and receives the updated list HTML. The `#todo-list` content is replaced. The form is cleared by the `htmx:afterRequest` event.

3 - Mark as Completed: The user clicks the checkbox. HTMX makes a `POST` to `/todos/toggle`. The server updates the database, fetches all tasks and renders `todo-list.html` with the new data. The list is updated and the checkbox reflects the change.

4 - Delete Task: The user clicks the icon to delete. HTMX makes a `DELETE` to `/todos/delete`. The server removes it from the database and returns the list without the deleted task.

#### See how our little project turned out

![project run](hgfhfgh456.gif)

## The Database

Persistence is done with SQLite.

```go
package db

import (
	"database/sql"
	"log"

	_ "modernc.org/sqlite"
)

func InitDB() *sql.DB {
  db, err := sql.Open("sqlite", "todos.db");
 	if err != nil {
		log.Fatalf("Erro ao abrir banco de dados: %v", err)
	}

	db.SetMaxOpenConns(1)

	query := `
	CREATE TABLE IF NOT EXISTS todos (
		id INTEGER PRIMARY KEY AUTOINCREMENT,
		title TEXT NOT NULL,
		completed BOOLEAN NOT NULL DEFAULT 0
	);`

	_, err = db.Exec(query)
	if err != nil {
		log.Fatalf("Erro ao criar tabela: %v", err)
	}

	return db
}
```

## Not Everything Is Perfect: Cons of Using HTMX

**1 - Everything by Hand**: Unlike React or Vue, where you have a huge ecosystem of libraries and ready-made components (`npm install` and ready), with HTMX you need to build every interaction from scratch. There is no `react-select` or `vue-datepicker` to import — either you write the entire component in Go templates or you don't have it.

**2 - Manual State Management:**: Modern frameworks have robust solutions for global state (Redux, Context API). With HTMX, the state lives only on the server. This is great until you need to synchronize multiple parts of the interface without reloading everything. Then you end up creating hacks in the HTML or making more requests than desired.

**3 - Slower Dev Experience**: No Hot Module Replacement, no `vite` reloading only what changed. Every change in the template requires a browser refresh. For those used to the fluidity of `npm run dev` in React, this weighs on daily work.

**4 - Limit for Complex Applications**: TMX shines in CRUDs and dashboards. But if you need a highly interactive interface — complex drag and drop, real-time editing, dynamic charts with rich local state — the "server-first" approach starts to struggle. You end up writing JavaScript anyway, just in a less structured way.

**5 - Total Server Dependency:**: Every click, every toggle, every filter makes an HTTP request. On slow or unstable connections, the user experience suffers. Modern SPAs can cache data and work offline; with HTMX, if the server goes down, the interface stops.

**6 - Limited Reactivity:**: There is no automatic two-way binding. If a form field needs to update another screen element in real time while the user types, you need to configure HTMX events manually. In Vue or React, this is `v-model` or `useState` and done.

## Conclusion

HTMX fits naturally into the Go ecosystem. Instead of creating a separate REST API and a frontend in React or Vue, you keep everything on the server: routes, business logic, HTML templates. HTMX adds the interactivity layer in a declarative way, directly in the markup.

This approach is ideal for internal applications, dashboards, CRUDs and any project where simplicity and development speed are priorities. You write less code, have fewer layers to maintain and still deliver a modern experience to the user.

But it is important to be clear that HTMX is not a silver bullet. If you are building something that requires a lot of client-side interactivity, complex shared state between components or needs to work with little server dependency, frameworks like React, Vue or Angular are still the most appropriate choice. The key is to choose the right tool for the right problem and for simple and direct applications, HTMX + Go is a powerful and productive combination.

## Useful Links

- [Repository](https://github.com/wiliamvj/htmx) of the example
- [HTMX Documentation](https://htmx.org/docs/)
- [Templates in Go](https://pkg.go.dev/html/template)
