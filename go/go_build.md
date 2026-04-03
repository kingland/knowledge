Here's how to build Go projects with a `cmd/` folder layout:

## Standard `cmd/` Project Structure

```
myproject/
├── go.mod
├── cmd/
│   ├── myapp/
│   │   └── main.go
│   └── myother/
│       └── main.go
├── internal/
└── pkg/
```

The `cmd/` directory holds the main applications. The directory name for each application should match the name of the executable you want (e.g., `/cmd/myapp`).

---

## Build Commands

**Build a specific app in `cmd/`:**
```bash
# From project root
go build ./cmd/myapp

# With explicit output name
go build -o bin/myapp ./cmd/myapp
```

**Build all apps in `cmd/` at once:**
```bash
go build ./cmd/...
```

**Build all into a `bin/` folder:**
```bash
go build -o bin/ ./cmd/...
```

**Run directly without building a file:**
```bash
go run ./cmd/myapp
```

---

## On Windows (MSVC environment)

```bat
REM Build single app
go build -o bin\myapp.exe .\cmd\myapp

REM Build all
go build -o bin\ .\cmd\...
```

---

## Cross-compile from `cmd/`

```bash
# Linux binary from Windows
GOOS=linux GOARCH=amd64 go build -o bin/myapp-linux ./cmd/myapp

# Windows binary from Linux
GOOS=windows GOARCH=amd64 go build -o bin/myapp.exe ./cmd/myapp
```

---

## Key Points

- Always run `go build` **from the project root** (where `go.mod` lives), using `./cmd/appname` as the path
- By default, `go build` generates a binary in the current directory, named after the package's directory
- Use `-o` flag to control output location and name
- `./cmd/...` wildcard builds **all** commands under `cmd/` in one shot