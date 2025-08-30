# Capítulo 11: Mastering Concurrent Processes with core.async

## 🎯 Conceito Central: Processes

**Process** = unidade de lógica que roda concorrentemente e responde a eventos

### Modelo Mental: Máquina de Hot Dog
```
Mundo Real: Entidades independentes respondem a eventos
Máquina de Hot Dog: recebe dinheiro → dispensa hot dog

Programa: Processos independentes respondem a eventos via channels
```

### Filosofia core.async
**Diferença**: Ao invés de tasks paralelas ou fire-and-forget
**Foco**: Processos que se comunicam e coordenam via channels

## 🚀 Setup e Primeiros Passos

### Configuração do Projeto
```clojure
;; project.clj
:dependencies [[org.clojure/clojure "1.9.0"]
               [org.clojure/core.async "0.1.346.0-17112a-alpha"]]

;; namespace
(ns my-project.core
  (:require [clojure.core.async :as a
             :refer [>! <! >!! <!! go chan buffer close! thread
                     alts! alts!! timeout]]))
```

### Primeiro Processo
```clojure
;; Criando channel
(def echo-chan (chan))

;; Criando processo que escuta
(go (println (<! echo-chan)))

;; Enviando mensagem
(>!! echo-chan "ketchup")
; => true
; => ketchup
```

### Como Funciona
1. **Channel**: Meio de comunicação entre processos
2. **go**: Cria processo assíncrono (go block)
3. **`<`**: take - pega mensagem do channel
4. **`>`**: put - coloca mensagem no channel
5. **Waiting**: Processos esperam put/take completar

## 📡 Channels e Comunicação

### Channels Básicos
```clojure
;; Channel simples - sem buffer
(def simple-chan (chan))

;; Este código BLOQUEIA porque não há listener
(>!! (chan) "mustard")  ; ⚠️ Bloqueia REPL!

;; Deve ter processo esperando
(def listener-chan (chan))
(go (println "Recebi:" (<! listener-chan)))
(>!! listener-chan "mensagem")  ; => Funciona!
```

### Buffering
```clojure
;; Buffer com tamanho 2
(def echo-buffer (chan 2))
(>!! echo-buffer "ketchup")    ; => true (não bloqueia)
(>!! echo-buffer "mustard")    ; => true (não bloqueia)  
(>!! echo-buffer "mayo")       ; => BLOQUEIA (buffer cheio)

;; Sliding buffer - FIFO (remove mais antigo)
(def sliding-buf (chan (sliding-buffer 2)))

;; Dropping buffer - LIFO (remove mais novo)  
(def dropping-buf (chan (dropping-buffer 2)))
```

### Analogia do Chef de Ketchup
```
Sem Buffer: Chef espera alguém pegar antes de fazer mais
Com Buffer: Chef tem prateleira para N batches
Sliding Buffer: Joga fora batch mais antigo quando cheio
Dropping Buffer: Joga fora batch mais novo quando cheio
```

## 🚗 Blocking vs Parking

### Tabela de Operações
| Operação | Dentro go block | Fora go block |
|----------|----------------|---------------|
| put      | `>!` ou `>!!`  | `>!!`         |
| take     | `<!` ou `<!!`  | `<!!`         |

### Conceitos
**Blocking (`>!!`, `<!!`)**:
- Thread para e espera
- Thread não faz nada
- Como I/O tradicional

**Parking (`>!`, `<!`)**:
- Thread liberada para outros processos
- Apenas dentro de go blocks
- Permite 1000 processos em poucos threads

### Exemplo: Eficiência com go blocks
```clojure
;; 1000 processos, poucos threads!
(def hi-chan (chan))
(doseq [n (range 1000)]
  (go (>! hi-chan (str "hi " n))))

;; Parking permite interleaving de processos numa thread
;; Processo A espera → Thread roda Processo B
;; Processo B espera → Thread volta para Processo A
```

### thread para Tarefas Longas
```clojure
;; Use thread para operações longas
(thread (println (<!! echo-chan)))
(>!! echo-chan "mustard")

;; thread retorna channel com resultado
(let [t (thread "chili")]
  (<!! t))
; => "chili"

;; REGRA: Use thread para operações que não podem "park"
;; (download de arquivos, processamento pesado)
```

## 🌭 Hot Dog Machine: Exemplo Prático

### Versão Simples
```clojure
(defn hot-dog-machine []
  (let [in (chan)          ; Canal para receber dinheiro
        out (chan)]        ; Canal para dispensar hot dog
    (go (<! in)            ; Espera dinheiro
        (>! out "hot dog")) ; Dispensa hot dog
    [in out]))             ; Retorna canais

;; Usando a máquina
(let [[in out] (hot-dog-machine)]
  (>!! in "pocket lint")   ; Qualquer coisa funciona!
  (<!! out))               ; => "hot dog"
```

### Versão Robusta
```clojure
(defn hot-dog-machine-v2 [hot-dog-count]
  (let [in (chan)
        out (chan)]
    (go (loop [hc hot-dog-count]
          (if (> hc 0)
            (let [input (<! in)]
              (if (= 3 input)                    ; Apenas $3
                (do (>! out "hot dog")
                    (recur (dec hc)))
                (do (>! out "wilted lettuce")    ; Troco errado
                    (recur hc))))
            (do (close! in)                      ; Sem hot dogs
                (close! out)))))
    [in out]))

;; Testando
(let [[in out] (hot-dog-machine-v2 2)]
  (>!! in "pocket lint")
  (println (<!! out))      ; => "wilted lettuce"
  
  (>!! in 3)
  (println (<!! out))      ; => "hot dog"
  
  (>!! in 3) 
  (println (<!! out))      ; => "hot dog"
  
  (>!! in 3)               ; Máquina fechada
  (<!! out))               ; => nil
```

### Pipeline de Processos
```clojure
;; Conectar canais: out de um = in do próximo
(let [c1 (chan)
      c2 (chan) 
      c3 (chan)]
  (go (>! c2 (clojure.string/upper-case (<! c1))))  ; MAIÚSCULA
  (go (>! c3 (clojure.string/reverse (<! c2))))     ; REVERSO
  (go (println (<! c3)))                            ; PRINT
  (>!! c1 "redrum"))
; => MURDER
```

## ⚡ alts!!: Primeiro Canal Disponível

### Uso Básico
```clojure
(defn upload [headshot c]
  (go (Thread/sleep (rand 100))  ; Simula upload
      (>! c headshot)))

(let [c1 (chan)
      c2 (chan)  
      c3 (chan)]
  (upload "serious.jpg" c1)
  (upload "fun.jpg" c2)
  (upload "sassy.jpg" c3)
  (let [[headshot channel] (alts!! [c1 c2 c3])]
    (println "Notification for" headshot)))
; => Notification for sassy.jpg (primeiro a completar)
```

### Timeout
```clojure
(let [c1 (chan)]
  (upload "serious.jpg" c1)
  (let [[headshot channel] (alts!! [c1 (timeout 20)])]
    (if headshot
      (println "Sending notification for" headshot)
      (println "Timed out!"))))
; => Timed out!
```

### Put Operations
```clojure
(let [c1 (chan)
      c2 (chan)]
  (go (<! c2))  ; Processo esperando em c2
  (let [[value channel] (alts!! [c1 [c2 "put!"]])]
    (println value)     ; => true (put sucedeu)
    (= channel c2)))    ; => true
```

## 📋 Queues: Coordenação de Trabalho

### Exemplo: Download de Quotes
```clojure
(defn append-to-file [filename s]
  (spit filename s :append true))

(defn format-quote [quote]
  (str "=== BEGIN QUOTE ===\n" quote "=== END QUOTE ===\n\n"))

(defn random-quote []
  (format-quote (slurp "http://www.braveclojure.com/random-quote")))

(defn snag-quotes [filename num-quotes]
  (let [c (chan)]
    ;; Processo consumidor (escreve no arquivo)
    (go (while true (append-to-file filename (<! c))))
    
    ;; Processos produtores (buscam quotes)
    (dotimes [n num-quotes] 
      (go (>! c (random-quote))))))

;; Uso
(snag-quotes "quotes" 2)
;; Arquivo 'quotes' terá 2 quotes em ordem de chegada
```

### Diferença de Chapter 9
- **Chapter 9**: Ordem de criação das tasks
- **core.async**: Ordem de completion das tasks
- **Ambos**: Garantem escrita sequencial (não intercalada)

## 🔄 Process Pipelines vs Callback Hell

### Problema: Callback Hell (JavaScript)
```javascript
// Dependências complexas, estado compartilhado
getData(function(a) {
  getMoreData(a, function(b) {
    getEvenMoreData(b, function(c) {
      // ... nightmare continues
    });
  });
});
```

### Solução: Process Pipeline
```clojure
(defn upper-caser [in]
  (let [out (chan)]
    (go (while true (>! out (clojure.string/upper-case (<! in)))))
    out))

(defn reverser [in]  
  (let [out (chan)]
    (go (while true (>! out (clojure.string/reverse (<! in)))))
    out))

(defn printer [in]
  (go (while true (println (<! in)))))

;; Conectando pipeline
(def in-chan (chan))
(def upper-caser-out (upper-caser in-chan))
(def reverser-out (reverser upper-caser-out))
(printer reverser-out)

;; Usando
(>!! in-chan "redrum")  ; => MURDER
(>!! in-chan "repaid")  ; => DIAPER
```

### Vantagens do Pipeline
- **Isolamento**: Cada processo é independente
- **Clareza**: Lógica isolada, fácil de raciocinar
- **Composição**: out de um = in do próximo
- **Como funções puras**: Entrada → Processamento → Saída

## 📚 Conceitos-Chave Resumidos

### 🚀 Core Functions
- **`chan`**: Cria channel (optionally buffered)
- **`go`**: Cria processo com parking (thread pool)
- **`thread`**: Cria processo com blocking (nova thread)
- **`>!` / `<!`**: Put/take com parking (só em go blocks)
- **`>!!` / `<!!`**: Put/take com blocking 
- **`close!`**: Fecha channel
- **`alts!` / `alts!!`**: Primeiro canal disponível

### 📡 Channel Types
- **Unbuffered**: Encontro direto entre put e take
- **Buffered**: N mensagens antes de bloquear
- **Sliding**: Remove oldest quando cheio
- **Dropping**: Remove newest quando cheio

### 🎯 Process Patterns
- **Producer-Consumer**: Um produz, outro consome
- **Pipeline**: Cadeia de transformações
- **Fan-out**: Um input → múltiplos processors
- **Fan-in**: Múltiplos inputs → um processor

### ⚖️ Blocking vs Parking
- **Parking**: Mais eficiente, permite muitos processos
- **Blocking**: Para operações longas que não podem parar
- **Thread pool**: go blocks compartilham poucos threads
- **Deadlock**: Cuidado com ciclos de dependência

## 💡 Melhores Práticas

### ✅ Do
- **Use go blocks** para processos leves e I/O
- **Use thread** para processamento pesado 
- **Close channels** quando terminar produção
- **Handle nil** de channels fechados
- **Pipeline design** para lógica clara
- **timeout** para operações com limite de tempo

### ❌ Don't
- **Misture blocking/parking** desnecessariamente
- **Esqueça close!** channels - pode causar resource leak
- **Use channels para tudo** - às vezes atoms são melhores
- **Crie deadlocks** com dependências circulares

### 🎯 Quando Usar core.async
- **Coordenação** entre processos independentes
- **Pipeline** de transformações
- **Event-driven** programming
- **Alternative** para callbacks
- **Producer-consumer** patterns

### 🚫 Quando NÃO Usar
- **Estado simples** (use atoms)
- **Cálculos paralelos** (use pmap)
- **One-shot tasks** (use future)

---

> **Próximo**: Capítulo 12 - Working with the JVM

> **Lição Principal**: core.async = processos independentes + channels de comunicação. Pipeline > Callback Hell!