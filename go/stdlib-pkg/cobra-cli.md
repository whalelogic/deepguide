
# 🐍 CLI Development with Go and Cobra



**Cobra** is a widely used Go library for creating powerful **command-line interfaces (CLIs)**. It provides a structured application model, automatic help generation, command hierarchies, and flag parsing. This guide explains how Cobra works, how to build a production‐grade CLI, and includes a detailed reference table of common functions.



---

## 1. Project Initialization

### Install Cobra
```bash
go get -u github.com/spf13/cobra@latest
```

### Initialize a Cobra Application
```bash
cobra-cli init myapp
```

This creates:
- `cmd/root.go` – root command definition  
- `main.go` – application entrypoint  
- Proper folder structure for scalable CLIs  

---

## 2. Creating Commands

### Add a New Command
```bash
cobra-cli add serve
```

This generates `cmd/serve.go` with a skeleton command.

### Example: Implementing a Serve Command
```go
var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the HTTP server",
    Long:  "Bootstraps and starts an HTTP server with routing, middleware, and configuration.",
    Run: func(cmd *cobra.Command, args []string) {
        port, _ := cmd.Flags().GetInt("port")
        log.Printf("Starting server on port %d...", port)
    },
}

func init() {
    rootCmd.AddCommand(serveCmd)
    serveCmd.Flags().IntP("port", "p", 8080, "Port to run the server on")
}
```

---

## 3. Flags

### Types of Flags
- Persistent Flags: apply to a command and its descendants  
- Local Flags: only for the current command  

### Persistent Flag Example
```go
rootCmd.PersistentFlags().String("config", "", "Path to configuration file")
```

### Local Flag Example
```go
serveCmd.Flags().Bool("debug", false, "Enable debug logging")
```

---

## 4. Command Hierarchy (Parent/Child Commands)

Cobra allows complex CLI structures:

```
myapp
 ├── serve
 ├── user
 │    ├── add
 │    ├── delete
 │    └── list
 └── version
```

Example: Adding user subcommands
```bash
cobra-cli add user
cobra-cli add add --parent user
```

---

## 5. Auto-Generated Help and Docs

Cobra auto-generates:
- `--help` output  
- command usage  
- flag descriptions  

You can also export Markdown docs:
```go
import "github.com/spf13/cobra/doc"

doc.GenMarkdownTree(rootCmd, "./docs")
```

---

## 6. Execution Model

The entrypoint calls:
```go
func Execute() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

Execution proceeds through:
1. Argument parsing  
2. Flag binding  
3. Validation  
4. Running the attached `Run()` function  

---

## 7. Comprehensive Cobra Function Reference

| Function | Real-World Use Case | Example | Why | How | What |
|---------|---------------------|---------|------|------|-------|
| `AddCommand()` | Build complex CLI hierarchies | `rootCmd.AddCommand(serveCmd)` | Attach subcommands | Call on parent command | Registers a child command |
| `Flags()` | Add local flags | `cmd.Flags().Bool("debug", false, "debug mode")` | Command config | Define before execute | A flag set for the command |
| `PersistentFlags()` | Global/shared flags | `rootCmd.PersistentFlags().String("config", "", "")` | Configuration passed once | Declare on root or parent | Flags inherited by subcommands |
| `MarkFlagRequired()` | Validate required flags | `cmd.MarkFlagRequired("port")` | Enforce correctness | Call after flag creation | Marks flag as mandatory |
| `Run` field | Main command logic | `Run: func(cmd, args){...}` | Define behavior | Inline function | Executes when command runs |
| `RunE` | Error-aware command logic | `RunE: func(cmd, args) error {...}` | Propagate errors | Return error | Error-handling version of Run |
| `Use` | Defines command invocation | `Use: "serve"` | CLI UX clarity | string identifier | Name used in CLI |
| `Short` | One-line help description | `Short: "Start server"` | Improve help UX | Provide summary | Shown in help menus |
| `Long` | Extended help text | `Long: "Starts and configures server"` | Document intent | Multi-line description | Detailed command docs |
| `SilenceErrors` | Suppress automatic error printing | `cmd.SilenceErrors = true` | Custom error handling | Set to true | CLI output control |
| `TraverseChildren` | Allow implicit command traversal | `cmd.TraverseChildren = true` | Nested command shortcuts | Set true | Auto-walk command tree |

---

## 8. Full Example CLI (Production Pattern)

```go
package main

import (
    "log"
    "os"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "A modern CLI application built with Cobra",
}

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Run the HTTP server",
    RunE: func(cmd *cobra.Command, args []string) error {
        port, _ := cmd.Flags().GetInt("port")
        log.Printf("Starting server on port %d", port)
        return nil
    },
}

func init() {
    serveCmd.Flags().IntP("port", "p", 8080, "Port to run the server on")
    rootCmd.AddCommand(serveCmd)
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

I Hope you found this useful!

---
