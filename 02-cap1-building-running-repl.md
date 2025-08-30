# Capítulo 1: Building, Running e o REPL

## 🎯 Objetivos do Capítulo
- Configurar ambiente básico para Clojure
- Criar, construir e executar programas
- Usar o REPL para experimentação
- Entender a relação Clojure/JVM

## 🔍 O Que É Clojure?

### Definições Fundamentais
- **Clojure (linguagem)**: Dialeto Lisp com foco funcional
- **Clojure (compilador)**: JAR executável `clojure.jar`
- **Linguagem hospedada**: Roda dentro da JVM

### Características Principais
- ✅ Sintaxe Lisp expressiva
- ✅ Programação funcional 
- ✅ Ferramentas para concorrência
- ✅ Acesso ao ecossistema Java

### Como Funciona
```
Código Clojure → clojure.jar → Bytecode Java → JVM
```

1. JVM executa bytecode Java
2. `clojure.jar` lê código Clojure
3. Produz bytecode Java
4. Mesmo processo JVM executa o bytecode

## 🛠️ Leiningen - Build Tool

### Instalação
1. Instalar Java 1.6+ (`java -version`)
2. Instalar Leiningen ([leiningen.org](http://leiningen.org/))
3. Leiningen baixa automaticamente `clojure.jar`

### Tarefas Principais
- Criar projetos novos
- Executar código Clojure  
- Construir JARs executáveis
- Abrir REPL

## 📁 Estrutura de Projeto

### Criando Projeto
```bash
lein new app clojure-noob
```

### Estrutura Resultante
```
clojure-noob/
├── project.clj          # Configuração do projeto
├── src/
│   └── clojure_noob/
│       └── core.clj      # Código principal
├── test/                 # Testes
├── resources/           # Assets (imagens, etc)
├── README.md
└── .gitignore
```

### Arquivos Importantes
- **`project.clj`**: Configuração Leiningen (dependências, entry point)
- **`src/clojure_noob/core.clj`**: Onde escrever código
- **`test/`**: Diretório de testes
- **`resources/`**: Assets do projeto

## ⚡ Executando Código

### Código Inicial (`core.clj`)
```clojure
(ns clojure-noob.core
  (:gen-class))

(defn -main
  "I don't do a whole lot...yet."
  [& args]
  (println "Hello, World!"))
```

### Executar Durante Desenvolvimento
```bash
lein run
```

### Construir JAR Executável
```bash
lein uberjar
```
- Cria: `target/uberjar/clojure-noob-0.1.0-SNAPSHOT-standalone.jar`

### Executar JAR
```bash
java -jar target/uberjar/clojure-noob-0.1.0-SNAPSHOT-standalone.jar
```

## 🔄 REPL (Read-Eval-Print Loop)

### O Que É?
- **R**ead: Lê entrada do usuário
- **E**val: Avalia/executa o código  
- **P**rint: Imprime resultado
- **L**oop: Volta ao início

### Iniciando REPL
```bash
lein repl
```

### Saída Típica
```
nREPL server started on port 28925
REPL-y 0.1.10  
Clojure 1.9.0
clojure-noob.core=>
```

### Exemplos de Uso
```clojure
clojure-noob.core=> (-main)
I'm a little teapot!
nil

clojure-noob.core=> (+ 1 2 3 4)
10

clojure-noob.core=> (* 1 2 3 4)  
24

clojure-noob.core=> (first [1 2 3 4])
1
```

### Características do REPL
- ✅ Feedback imediato
- ✅ Experimentação rápida
- ✅ Desenvolvimento interativo
- ✅ Pode conectar a aplicações em produção
- ✅ Essencial para desenvolvimento Lisp

## 🎨 Sintaxe Básica

### Notação Prefixa
```clojure
; Ao invés de: 1 + 2 + 3
; Clojure usa: (+ 1 2 3)

(operador arg1 arg2 arg3)
```

### Exemplos
```clojure
(+ 1 2)        ; => 3
(* 3 4 5)      ; => 60  
(- 10 2)       ; => 8
(/ 20 4)       ; => 5
```

## 📝 Editores Recomendados

### Emacs (Mais Popular)
- Integração tight com REPL
- Bem adaptado para Lisp
- Capítulo 2 do livro

### Alternativas
- **Sublime Text 2**: [Tutorial no YouTube](http://www.youtube.com/watch?v=wBl0rYXQdGg/)
- **Vim**: [Artigo setup](http://mybuddymichael.com/writings/writing-clojure-with-vim-in-2013.html)
- **Eclipse**: Plugin Counterclockwise
- **IntelliJ**: Cursive Clojure IDE
- **Nightcode**: IDE simples escrita em Clojure

## 💡 Conceitos-Chave

### Fluxo de Desenvolvimento
1. **Escrever código** em `src/`
2. **Testar no REPL** (`lein repl`)
3. **Executar aplicação** (`lein run`)
4. **Construir JAR** (`lein uberjar`)
5. **Distribuir JAR** (roda em qualquer JVM)

### Vantagens do REPL
- Ciclo de feedback rápido
- Exploração interativa
- Desenvolvimento incremental
- Debug em tempo real
- Parte essencial da experiência Lisp

### Relacionamento JVM
- Clojure compila para bytecode Java
- Aproveita otimizações da JVM
- Acessa bibliotecas Java
- Threading e garbage collection da JVM

## 🎯 Próximos Passos

1. **Praticar no REPL**: Experimentar com operações básicas
2. **Modificar código**: Alterar função `-main`
3. **Testar build**: Criar e executar JAR
4. **Preparar editor**: Configurar ambiente de desenvolvimento

---

> **Próximo**: Capítulo 2 - Como Usar Emacs

> **Dica**: Use o REPL constantemente - é fundamental para o aprendizado de Clojure!