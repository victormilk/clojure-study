# Capítulo 12: Working with the JVM

## 🎯 Conceito Central: Clojure na JVM

### Por Que Clojure na JVM?
**Vantagens**:
1. **Mesma execução** que programas Java
2. **Acesso a objetos Java** para funcionalidades core
3. **Ecossistema gigante** de bibliotecas Java disponível

**Analogia**: Clojure = comunidade utópica num país distópico
- Prefere interagir com outros utópicos (Clojure code)
- Às vezes precisa falar com locais (Java) para fazer as coisas

## 🔧 Como a JVM Funciona

### Computador Normal
```
Código C++ → Compilador → Instruções máquina → CPU executa
```

### JVM (Máquina Virtual)
```
Código Java → javac → Bytecode Java → JVM → Instruções máquina → CPU
                ↑
    Clojure também produz bytecode Java!
```

### Características JVM
- **Virtual machine**: Tradução via software, não hardware
- **Just-in-time compilation**: Bytecode → código máquina on-the-fly
- **Language agnostic**: Scala, JRuby, Clojure, Java → mesmo bytecode
- **JAR files**: Pacotes de arquivos .class

## 📝 Java Básico: OOP em 2 Minutos

### Classes, Objetos, Métodos
```java
// Classe = fábrica de androids
// Objeto = android individual  
// Método = comando que android entende

ScaryClown bellyRubsTheClown = new ScaryClown();  // Criar objeto
bellyRubsTheClown.balloonCount();                 // => 0
bellyRubsTheClown.receiveBalloons(2);             // Método com argumento  
bellyRubsTheClown.balloonCount();                 // => 2

// Método de classe (static)
Math.abs(-50);  // => 50
```

## 🏴‍☠️ Exemplo Java: Pirate Phrases

### Programa Simples
```java
// PiratePhrases.java
public class PiratePhrases {
    public static void main(String[] args) {
        System.out.println("Shiver me timbers!!!");
    }
}
```

### Compilação e Execução
```bash
javac PiratePhrases.java    # Cria PiratePhrases.class
java PiratePhrases          # Executa (busca main method)
```

### Como Java Funciona
1. **javac** encontra classe no classpath
2. **Classpath** = lista de diretórios para buscar classes
3. **Nome arquivo** deve coincidir com nome da classe
4. **public main** é ponto de entrada obrigatório

## 📦 Packages e Imports

### Packages = Namespaces Java
```java
// pirate_phrases/Greetings.java
package pirate_phrases;

public class Greetings {
    public static void hello() {
        System.out.println("Shiver me timbers!!!");
    }
}
```

### Imports = Evitar Prefixo
```java
// PirateConversation.java
import pirate_phrases.*;  // Importa todas classes do package

public class PirateConversation {
    public static void main(String[] args) {
        Greetings greetings = new Greetings();  // Sem prefixo!
        greetings.hello();
    }
}
```

### Estrutura de Diretórios
```
phrasebook/
├── PirateConversation.java
└── pirate_phrases/
    ├── Greetings.java
    └── Farewells.java
```

**REGRA**: Package names → directory structure

## 🗃️ JAR Files

### Bundling Classes
```bash
jar cvfe conversation.jar PirateConversation PirateConversation.class pirate_phrases/*.class
java -jar conversation.jar
```

### JAR = ZIP + Manifest
```
conversation.jar/
├── META-INF/
│   └── MANIFEST.MF      # Main-Class: PirateConversation
├── PirateConversation.class
└── pirate_phrases/
    ├── Greetings.class
    └── Farewells.class
```

## 🔍 clojure.jar Internamente

### Como Clojure Roda
```bash
java -jar clojure-1.9.0.jar  # Inicia REPL
```

### META-INF/MANIFEST.MF
```
Main-Class: clojure.main
```

### clojure/main.java (GitHub)
```java
package clojure;

public class main {
    public static void main(String[] args) {
        REQUIRE.invoke(CLOJURE_MAIN);
        MAIN.applyTo(RT.seq(args));
    }
}
```

**Resultado**: Clojure é programa JVM como qualquer outro!

## 🚀 Clojure App JARs

### (:gen-class) Directive
```clojure
(ns my-app.core
  (:gen-class))  ; Gera classe Java para namespace

(defn -main [& args]  ; main method
  (println "Hello from Clojure!"))
```

### project.clj
```clojure
:main ^:skip-aot my-app.core
```

### Build e Run
```bash
lein uberjar
java -jar target/uberjar/my-app-standalone.jar
```

**Processo**: Clojure namespace → classe Java → JAR executável

## ⚡ Java Interop Syntax

### Chamando Métodos em Objetos
```clojure
;; (.methodName object args)
(.toUpperCase "By Bluebeard's bananas!")
; => "BY BLUEBEARD'S BANANAS!"

(.indexOf "Let's synergize our bleeding edges" "y")  
; => 7

;; Equivale ao Java:
;; "string".toUpperCase()
;; "string".indexOf("y")
```

### Métodos e Fields Estáticos
```clojure
;; (ClassName/methodName args)
(java.lang.Math/abs -3)    ; => 3

;; ClassName/FIELD  
java.lang.Math/PI          ; => 3.141592653589793
```

### Dot Special Form (Macro Expansion)
```clojure
(macroexpand-1 '(.toUpperCase "hello"))
; => (. "hello" toUpperCase)

(macroexpand-1 '(Math/abs -3))  
; => (. Math abs -3)

;; Forma geral: (. object-or-class method-or-field args*)
```

## 🏗️ Criando e Mutando Objetos

### Criação de Objetos
```clojure
;; Duas formas equivalentes
(new String)                ; => ""
(String.)                   ; => "" (preferida)

(String. "Hello world")     ; => "Hello world"
```

### Objetos Mutáveis: java.util.Stack
```clojure
;; Stack MUTABLE (diferente de Clojure data structures)
(let [stack (java.util.Stack.)]
  (.push stack "First item")
  (.push stack "Second item") 
  stack)
; => ["First item" "Second item"]

;; Pode usar seq functions para LEITURA
(let [stack (java.util.Stack.)]
  (.push stack "Item")
  (first stack))  ; => "Item"

;; MAS NÃO para modificação
;; (conj stack "item") ; => Exception!
```

### doto Macro: Múltiplas Operações
```clojure
;; Sem doto - verboso
(let [stack (java.util.Stack.)]
  (.push stack "First")
  (.push stack "Second")
  stack)

;; Com doto - conciso  
(doto (java.util.Stack.)
  (.push "First")
  (.push "Second"))
; => ["First" "Second"]

;; Macro expansion:
;; (let [obj (java.util.Stack.)]
;;   (.push obj "First")
;;   (.push obj "Second") 
;;   obj)
```

## 📥 Importing Classes

### Import Simples
```clojure
(import java.util.Stack)
(Stack.)  ; => []
```

### Import Múltiplo
```clojure
(import [java.util Date Stack]
        [java.net Proxy URI])

(Date.)  ; => #inst "2025-01-15T10:30:00.000-00:00"
```

### Import no Namespace (Preferido)
```clojure
(ns pirate.talk
  (:import [java.util Date Stack]
           [java.net Proxy URI]))
```

**Auto-import**: Classes em `java.lang` (String, Math, etc.)

## 🔧 Classes Java Comuns

### System Class
```clojure
;; Environment variables
(System/getenv)
; => {"USER" "john", "JAVA_ARCH" "x86_64", ...}

;; JVM properties
(System/getProperty "user.dir")     ; => "/Users/john/project"
(System/getProperty "java.version") ; => "1.8.0_40"

;; Exit program
(System/exit 0)  ; Status code 0 = success
```

### Date Class
```clojure
;; Date literals
#inst "2025-01-15T10:30:00.000-00:00"

;; Creating dates
(java.util.Date.)  ; => Current time

;; Para manipulação complexa: use clj-time library
;; https://github.com/clj-time/clj-time
```

## 📁 Files e Input/Output

### Java IO é Complexo
**Problema**: Classes separadas para:
- **Properties**: `java.io.File` (exists? canWrite? getPath?)
- **Reading**: `java.io.BufferedReader`, `java.io.FileReader`  
- **Writing**: `java.io.BufferedWriter`, `java.io.FileWriter`

### Clojure Simplifica: spit e slurp
```clojure
;; Escrever arquivo
(spit "/tmp/hercules-todo-list"
      "- kill dat lion brov\n- chop up hydra")

;; Ler arquivo
(slurp "/tmp/hercules-todo-list")
; => "- kill dat lion brov\n- chop up hydra"

;; Funciona com diferentes resources
(let [s (java.io.StringWriter.)]
  (spit s "- capture cerynian hind")
  (.toString s))
; => "- capture cerynian hind"

(let [s (java.io.StringReader. "- get erymanthian pig")]
  (slurp s))
; => "- get erymanthian pig"
```

### with-open: Resource Management
```clojure
;; Fecha resource automaticamente
(with-open [todo-rdr (clojure.java.io/reader "/tmp/hercules-todo-list")]
  (println (first (line-seq todo-rdr))))
; => - kill dat lion brov

;; Equivale a try-with-resources em Java
```

### File Properties
```clojure
(let [file (java.io.File. "/")]
  (println (.exists file))     ; => true
  (println (.canWrite file))   ; => false  
  (println (.getPath file)))   ; => /
```

## 📚 Conceitos-Chave Resumidos

### 🏗️ JVM Architecture
- **Bytecode**: Linguagem intermediária da JVM
- **JAR files**: Pacotes de classes (.zip com manifest)
- **Classpath**: Onde JVM procura classes
- **Entry point**: main method para execução

### 🔧 Java Interop Syntax
- **Object methods**: `(.method object args)`
- **Static methods**: `(Class/method args)`
- **Static fields**: `Class/FIELD`
- **Object creation**: `(Class. args)` ou `(new Class args)`
- **Multiple operations**: `doto` macro

### 📦 Organization
- **Packages**: Organização de código (= namespaces)
- **Imports**: Evitar prefixos longos
- **Directory structure**: Package name → filesystem path
- **Clojure namespaces**: `(:import)` no ns declaration

### 🔄 Clojure ↔ Java
- **(:gen-class)**: Gera classe Java para namespace
- **-main function**: Entry point do programa
- **lein uberjar**: Cria JAR executável standalone
- **Auto-import**: java.lang.* classes

## 💡 Melhores Práticas

### ✅ Do  
- **Import no ns**: Declare imports no namespace declaration
- **Use doto**: Para múltiplas operações no mesmo objeto
- **spit/slurp**: Para I/O simples ao invés de classes Java
- **with-open**: Para resource management
- **System properties**: Para configuração e info ambiente

### ❌ Don't
- **Misturar mutação**: Objetos Java mutáveis quebram paradigma funcional
- **Esquecer imports**: java.lang auto-imported, resto precisa import
- **Classpath errado**: Verificar estrutura de diretórios
- **Resource leak**: Sempre fechar resources (use with-open)

### 🎯 Quando Usar Java Interop
- **I/O operations**: Files, network, database
- **System integration**: Environment, processes
- **Third-party libraries**: Quando não há alternativa Clojure
- **Performance critical**: Ocasionalmente para otimização

### 🚫 Quando Evitar
- **Estado mutável**: Prefira atoms/refs quando possível
- **Programação OOP**: Use abstrações funcionais do Clojure
- **Strings/Math**: Clojure já tem boas abstrações

---

> **Próximo**: Capítulo 13 - Multimethods, Protocols, and Records

> **Lição Principal**: JVM = plataforma robusta. Interop = ponte entre mundos. Use quando necessário, prefira Clojure quando possível!