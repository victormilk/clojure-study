# Capítulo 8: Writing Macros

## 🎯 Conceito Central: Macros São Essenciais

**Macros não são "feature esotérica"** - eles são fundamentais no Clojure. Muitas operações que você considera "built-in" são implementadas como macros.

### when é uma Macro!
```clojure
;; Você acha que when é especial? É macro!
(macroexpand '(when true (println "Executado")))
; => (if true (do (println "Executado")))

;; when é implementado assim:
(defmacro when [test & body]
  (list 'if test (cons 'do body)))
```

## 🔧 Anatomia de uma Macro

### Estrutura Básica
```clojure
(defmacro nome-macro
  "Docstring opcional"
  [argumentos]
  corpo-que-retorna-lista)
```

### Exemplo: infix
```clojure
(defmacro infix
  "Notação infix para quem tem saudade da infância"
  [infixed]
  (list (second infixed) (first infixed) (last infixed)))

;; Uso
(infix (1 + 1))    ; => 2
;; Expande para: (+ 1 1)

;; Com destructuring
(defmacro infix-2
  [[operando1 op operando2]]
  (list op operando1 operando2))
```

### Diferença Crucial: Função vs Macro
```clojure
;; FUNÇÃO: argumentos são avaliados antes de passar
(defn funcao-exemplo [x] (println x))
(funcao-exemplo (+ 1 2))  ; (+ 1 2) é avaliado → 3

;; MACRO: argumentos passados sem avaliar
(defmacro macro-exemplo [x] `(println '~x))
(macro-exemplo (+ 1 2))  ; (+ 1 2) vai literal para a macro
```

## 🏗️ Construindo Listas para Avaliação

### Distinção Fundamental: Símbolos vs Valores

**Problema comum**:
```clojure
;; ERRADO - tenta obter VALOR dos símbolos
(defmacro my-print-whoopsie [expression]
  (list let [result expression]
        (list println result)
        result))
; => Exception: Can't take the value of a macro: #'clojure.core/let
```

**Solução - Quote os símbolos**:
```clojure
;; CORRETO - usa SÍMBOLOS, não valores
(defmacro my-print [expression]
  (list 'let ['result expression]
        (list 'println 'result)
        'result))

;; Uso
(my-print (+ 1 2))
;; Expande para:
;; (let [result (+ 1 2)]
;;   (println result)
;;   result)
```

### Simple Quoting
```clojure
;; Sem quote - avalia
(+ 1 2)                    ; => 3

;; Com quote - retorna estrutura
(quote (+ 1 2))           ; => (+ 1 2)
'(+ 1 2)                  ; => (+ 1 2) (shorthand)

;; Símbolos
sweating-to-the-oldies    ; => Exception (não existe)
'sweating-to-the-oldies   ; => sweating-to-the-oldies
```

## 🎨 Syntax Quoting: O Template System

### Diferenças do Quote Normal
```clojure
;; Quote normal - não resolve namespace
'+                         ; => +

;; Syntax quote - resolve namespace completo  
`+                         ; => clojure.core/+

;; Em listas
'(+ 1 2)                  ; => (+ 1 2)
`(+ 1 2)                  ; => (clojure.core/+ 1 2)
```

### Unquote: Escape do Template
```clojure
;; Sem unquote
`(+ 1 (inc 1))            ; => (clojure.core/+ 1 (clojure.core/inc 1))

;; Com unquote - avalia (inc 1)
`(+ 1 ~(inc 1))           ; => (clojure.core/+ 1 2)

;; Como interpolação de string
;; "Olá #{nome}!" em Ruby
;; `(println ~nome) em Clojure
```

### Comparação: list vs Syntax Quote
```clojure
;; Usando list - verboso
(list '+ 1 (inc 1))       ; => (+ 1 2)

;; Usando syntax quote - conciso e claro
`(+ 1 ~(inc 1))           ; => (clojure.core/+ 1 2)
```

## 🎯 Exemplo Prático: code-critic

### Versão com list (Verbosa)
```clojure
(defmacro code-critic
  [bad good]
  (list 'do
        (list 'println "Bad code:" (list 'quote bad))
        (list 'println "Good code:" (list 'quote good))))
```

### Versão com Syntax Quote (Limpa)
```clojure
(defmacro code-critic
  [bad good]
  `(do (println "Bad code:" (quote ~bad))
       (println "Good code:" (quote ~good))))
```

## ⚡ Refatorando com Unquote Splicing

### Problema: Duplicação
```clojure
;; Função helper
(defn criticize-code [criticism code]
  `(println ~criticism (quote ~code)))

;; Tentativa com map
(defmacro code-critic [bad good]
  `(do ~(map #(apply criticize-code %)
             [["Bad:" bad] ["Good:" good]])))
; => NullPointerException!
```

### Solução: Unquote Splicing (~@)
```clojure
;; Diferença crucial:
`(+ ~(list 1 2 3))        ; => (clojure.core/+ (1 2 3))
`(+ ~@(list 1 2 3))       ; => (clojure.core/+ 1 2 3)

;; Aplicando na macro
(defmacro code-critic [bad good]
  `(do ~@(map #(apply criticize-code %)
              [["Bad:" bad] ["Good:" good]])))
; => Funciona perfeitamente!
```

## ⚠️ Armadilhas das Macros

### 1. Variable Capture

**Problema**:
```clojure
(def message "Good job!")
(defmacro with-mischief [& stuff-to-do]
  (concat (list 'let ['message "Oh, big deal!"])
          stuff-to-do))

(with-mischief
  (println message))
; => "Oh, big deal!" (capturou nossa variável!)
```

**Solução: gensym**
```clojure
(defmacro without-mischief [& stuff-to-do]
  (let [macro-message (gensym 'message)]
    `(let [~macro-message "Oh, big deal!"]
       ~@stuff-to-do
       (println "Macro says:" ~macro-message))))

;; Auto-gensym - ainda mais fácil
(defmacro safe-macro [& stuff-to-do]
  `(let [message# "Oh, big deal!"]
     ~@stuff-to-do
     (println "Macro says:" message#)))
```

### 2. Double Evaluation

**Problema**:
```clojure
(defmacro report [to-try]
  `(if ~to-try
     (println (quote ~to-try) "success:" ~to-try)
     (println (quote ~to-try) "failure:" ~to-try)))

;; Problema: avalia duas vezes!
(report (do (Thread/sleep 1000) (+ 1 1)))
; => Demora 2 segundos! (executa sleep duas vezes)
```

**Solução: let binding**
```clojure
(defmacro report [to-try]
  `(let [result# ~to-try]
     (if result#
       (println (quote ~to-try) "success:" result#)
       (println (quote ~to-try) "failure:" result#))))
```

### 3. Macros All the Way Down

**Problema**: Macros não funcionam com runtime values
```clojure
;; Não funciona como esperado
(doseq [code ['(= 1 1) '(= 1 2)]]
  (report code))
; => "code was successful: (= 1 1)" (literal!)
```

**Solução**: Às vezes precisa de mais macros
```clojure
(defmacro doseq-macro [macroname & args]
  `(do ~@(map (fn [arg] (list macroname arg)) args)))

(doseq-macro report (= 1 1) (= 1 2))
; => Funciona, mas cuidado com complexidade!
```

## 🧪 Projeto Prático: Validação de Pedidos

### Setup dos Dados
```clojure
(def order-details
  {:name "João Silva"
   :email "joao.silvagmail.com"})  ; Email inválido (sem @)

(def order-details-validations
  {:name   ["Digite um nome" not-empty]
   :email  ["Digite um email" not-empty
            "Email inválido" #(re-seq #"@" %)]})
```

### Funções de Validação
```clojure
(defn error-messages-for
  "Retorna sequência de mensagens de erro"
  [to-validate message-validator-pairs]
  (map first 
       (filter #(not ((second %) to-validate))
               (partition 2 message-validator-pairs))))

(defn validate
  "Retorna map com vetor de erros para cada chave"
  [to-validate validations]
  (reduce (fn [errors validation]
            (let [[fieldname validation-groups] validation
                  value (get to-validate fieldname)
                  error-messages (error-messages-for value validation-groups)]
              (if (empty? error-messages)
                errors
                (assoc errors fieldname error-messages))))
          {}
          validations))

;; Teste
(validate order-details order-details-validations)
; => {:email ("Email inválido")}
```

### Macro if-valid
```clojure
;; Padrão comum repetitivo:
(let [errors (validate record validations)]
  (if (empty? errors)
    success-code
    failure-code))

;; Macro para abstrair o padrão:
(defmacro if-valid
  "Validação mais concisa"
  [to-validate validations errors-name & then-else]
  `(let [~errors-name (validate ~to-validate ~validations)]
     (if (empty? ~errors-name)
       ~@then-else)))

;; Uso
(if-valid order-details order-details-validations errors
  (render :success)
  (render :failure errors))

;; Expansão:
;; (let [errors (validate order-details order-details-validations)]
;;   (if (empty? errors)
;;     (render :success)
;;     (render :failure errors)))
```

## 📚 Conceitos-Chave Resumidos

### 🎨 Ferramentas Essenciais
- **Quote (')**: Previne avaliação → símbolos literais
- **Syntax Quote (`)**: Template com namespace resolution
- **Unquote (~)**: Escape dentro de syntax quote
- **Unquote Splicing (~@)**: Expande sequência
- **gensym**: Símbolos únicos para evitar captura

### ⚠️ Armadilhas Principais
- **Variable Capture**: Use gensym ou auto-gensym (#)
- **Double Evaluation**: Use let binding para avaliar uma vez
- **Macro Infection**: Cuidado para não precisar de macros infinitas

### 🎯 Quando Usar Macros
- **Nova sintaxe**: Quando precisa de controle de avaliação
- **DSLs**: Domain-specific languages
- **Abstração de padrões**: Como if-valid
- **Não**: Para comportamento que função pode fazer

## 💡 Melhores Práticas

### ✅ Do
- Use macros para **controle de avaliação** genuíno
- Abstraia **padrões repetitivos** que funções não conseguem
- Use `macroexpand` para debug
- Prefira **funções helper** dentro de macros
- Use **gensym** para símbolos internos

### ❌ Don't
- Use macros só por "parecer legal"
- Ignore variable capture e double evaluation
- Crie macros quando função seria suficiente
- Esqueça que macros só compõem com outras macros

## 🚀 Exercícios Recomendados

1. **when-valid**: Implemente como `when` mas para validação
2. **or macro**: Implemente `or` como macro (like `and`)
3. **defattrs**: Macro para criar múltiplas funções attr de uma vez

```clojure
;; Exercício 3 - exemplo de uso desejado:
(defattrs c-int :intelligence
          c-str :strength  
          c-dex :dexterity)
```

---

> **Próximo**: Capítulo 9 - Concurrent and Parallel Programming

> **Lição Principal**: Macros = Templates que controlam avaliação. Use com sabedoria, poder vem com responsabilidade!