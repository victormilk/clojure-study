# Capítulo 7: Clojure Alchemy - Reading, Evaluation e Macros

## 🎯 Conceito Central: Como Clojure Funciona

**Clojure opera em duas fases distintas:**
1. **Reader**: Transforma texto em estruturas de dados
2. **Evaluator**: Processa estruturas de dados para produzir resultados

## 📖 O Reader: Transformando Texto em Estruturas

### Processo do Reader
```
"(+ 1 2)" → Reader → (+ 1 2) → Evaluator → 3
```

O reader produz estruturas de dados, **não** código executável diretamente.

### Sintaxe Reader Macros
```clojure
;; String literal
"uma string"
; Reader macro para strings

;; Vector
[1 2 3]
; Reader macro que produz vetor

;; Map
{:nome "Ana" :idade 30}
; Reader macro que produz map

;; Set
#{:a :b :c}
; Reader macro que produz set

;; Quote
'(1 2 3)
; Equivale a: (quote (1 2 3))

;; Syntax quote
`(+ 1 ~x)
; Template para macros

;; Unquote
~variavel
; Dentro de syntax quote

;; Unquote-splicing
~@lista
; Expande lista dentro de syntax quote
```

### Reader Macros em Ação
```clojure
;; Deref reader macro
@meu-atom
; Equivale a: (deref meu-atom)

;; Function reader macro
#(+ % 1)
; Equivale a: (fn [%] (+ % 1))

;; Var reader macro
#'minha-funcao
; Equivale a: (var minha-funcao)

;; Regex reader macro
#"[0-9]+"
; Produz pattern regex
```

## 🔄 O Evaluator: Processando Estruturas

### Regras de Evaluation

**1. Símbolos**: Resolvem para valores que representam
```clojure
minha-var    ; Resolve para valor armazenado
```

**2. Listas**: Primeiro elemento como operador
```clojure
(+ 1 2)      ; + é operador, 1 2 são operandos
```

**3. Outros**: Avaliam para si mesmos
```clojure
42           ; => 42
"string"     ; => "string"
[1 2 3]      ; => [1 2 3]
```

### Evaluation de Listas Vazias e Especiais
```clojure
()           ; => ()
(if true 1 2); => 1 (forma especial)
(quote x)    ; => x (não avalia x)
```

## 🎨 Macros: Manipulando Código Como Dados

### Conceito Fundamental
Macros **transformam código** antes da evaluation:

```
Código → Macro → Novo Código → Evaluation → Resultado
```

### Exemplo Prático: when
```clojure
;; Macro when (simplificada)
(defmacro when [test & body]
  `(if ~test
     (do ~@body)))

;; Uso
(when true
  (println "Executado!")
  42)

;; Expande para:
(if true
  (do 
    (println "Executado!")
    42))
```

### Syntax Quoting para Macros
```clojure
;; Quote simples - não resolve símbolos
'(+ 1 2)     ; => (+ 1 2)

;; Syntax quote - resolve símbolos para namespace completo
`(+ 1 2)     ; => (clojure.core/+ 1 2)

;; Unquote - avalia dentro de syntax quote
(let [x 5]
  `(+ 1 ~x)) ; => (clojure.core/+ 1 5)

;; Unquote-splicing - expande sequência
(let [nums [2 3]]
  `(+ 1 ~@nums)) ; => (clojure.core/+ 1 2 3)
```

### Macro Simples: infix
```clojure
(defmacro infix [operando1 operador operando2]
  `(~operador ~operando1 ~operando2))

;; Uso
(infix 1 + 2)        ; => 3
;; Expande para: (+ 1 2)

;; Funciona com qualquer operador
(infix 10 / 2)       ; => 5
(infix "Olá" str " mundo") ; => "Olá mundo"
```

### Debugando Macros: macroexpand
```clojure
;; Ver expansão de macro
(macroexpand '(when true (println "teste")))
; => (if true (do (println "teste")))

(macroexpand '(infix 1 + 2))
; => (+ 1 2)

;; macroexpand-1 para expansão simples
(macroexpand-1 '(when true (println "teste")))
```

## 🔧 Criando Macros Úteis

### Macro unless (oposto de when)
```clojure
(defmacro unless [test & body]
  `(if (not ~test)
     (do ~@body)))

;; Uso
(unless false
  (println "Executado porque teste é false")
  :resultado)
; => :resultado
```

### Macro with-out-str personalizada
```clojure
(defmacro capture-output [& body]
  `(let [s# (java.io.StringWriter.)]
     (binding [*out* s#]
       ~@body
       (str s#))))

;; Uso
(capture-output
  (println "Linha 1")
  (println "Linha 2"))
; => "Linha 1\nLinha 2\n"
```

### Macro de Logging
```clojure
(defmacro log [level & messages]
  `(when (>= *log-level* ~level)
     (println ~@messages)))

;; Setup
(def ^:dynamic *log-level* 1)

;; Uso
(log 1 "DEBUG:" "valor x =" 42)
; => DEBUG: valor x = 42
```

## 🚨 Problemas Comuns com Macros

### 1. Captura de Símbolos
**Problema**:
```clojure
(defmacro broken-unless [test & body]
  `(let [resultado# (not ~test)]
     (if resultado# 
       (do ~@body))))
```

**Solução - Gensym**:
```clojure
(defmacro safe-unless [test & body]
  (let [resultado (gensym "resultado")]
    `(let [~resultado (not ~test)]
       (if ~resultado 
         (do ~@body)))))
```

### 2. Avaliação Múltipla
**Problema**:
```clojure
(defmacro bad-when [test & body]
  `(if ~test ~test ~@body)) ; Avalia test duas vezes!
```

**Solução**:
```clojure
(defmacro good-when [test & body]
  `(let [test-result# ~test]
     (if test-result#
       (do ~@body))))
```

## 🎯 Exemplos Avançados

### Macro para Benchmark
```clojure
(defmacro benchmark [& body]
  `(let [start# (System/nanoTime)
         result# (do ~@body)
         end# (System/nanoTime)
         time# (/ (- end# start#) 1e6)]
     (println "Tempo:" time# "ms")
     result#))

;; Uso
(benchmark
  (Thread/sleep 100)
  (+ 1 2 3))
; => Tempo: 103.2 ms
; => 6
```

### Macro para Try-Catch Simplificado
```clojure
(defmacro safely [& body]
  `(try
     ~@body
     (catch Exception e#
       (println "Erro:" (.getMessage e#))
       nil)))

;; Uso
(safely
  (/ 10 0))
; => Erro: Divide by zero
; => nil
```

## 📚 Conceitos-Chave Resumidos

### 🔄 Reader vs Evaluator
- **Reader**: Texto → Estruturas de dados
- **Reader macros**: Syntax shortcuts (`[]`, `{}`, `@`, etc.)
- **Evaluator**: Estruturas → Resultados
- **Separação clara**: Reader primeiro, depois evaluator

### 🎨 Macros Fundamentais
- **Template**: Syntax quote `` ` `` para estrutura
- **Interpolação**: Unquote `~` para valores
- **Splicing**: Unquote-splicing `~@` para listas
- **Hygiene**: Gensym para evitar conflitos

### 🔧 Ferramentas de Debug
- **macroexpand**: Ver expansão completa
- **macroexpand-1**: Ver um nível de expansão
- **gensym**: Criar símbolos únicos
- **Namespace qualification**: Syntax quote resolve símbolos

### ⚠️ Armadilhas Comuns
- **Symbol capture**: Use gensym ou auto-gensym (#)
- **Avaliação múltipla**: Let-bind resultados de expressões
- **Namespace leakage**: Cuidado com símbolos não qualificados
- **Overuse**: Nem tudo precisa ser macro

## 💡 Melhores Práticas

### ✅ Do
- Use macros para **nova sintaxe** que melhora expressividade
- Prefira funções quando **comportamento** é suficiente
- Teste expansões com `macroexpand`
- Use auto-gensym (`#`) quando possível
- Documente comportamento e expansão esperada

### ❌ Don't
- Não use macros apenas por performance
- Não ignore namespace hygiene
- Não faça macros que poderiam ser funções
- Não esqueça de testar com diferentes inputs
- Não torne a API mais complexa desnecessariamente

## 🎪 Exemplo Final: Macro DSL
```clojure
(defmacro defroutes [name & routes]
  (let [route-fns (map (fn [[method path handler]]
                        `(~method ~path ~handler))
                      (partition 3 routes))]
    `(def ~name
       {:routes [~@route-fns]})))

;; Uso
(defroutes api-routes
  :get "/users" get-users
  :post "/users" create-user
  :get "/health" health-check)

;; Expande para uma estrutura de dados com rotas
```

---

> **Próximo**: Capítulo 8 - Writing Macros

> **Lição Principal**: Macros = Transformação de código. Reader + Evaluator = Poder total sobre sintaxe!