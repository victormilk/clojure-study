# Capítulo 3: Do Things - Crash Course Clojure

## 🎯 O Que Aprenderemos

Este é o capítulo mais importante para iniciantes! Vamos aprender:
- **Sintaxe**: Como escrever código Clojure
- **Estruturas de dados**: Numbers, strings, maps, vectors, lists, sets
- **Funções**: Criar, chamar e usar funções
- **Projeto prático**: Modelar um hobbit e criar função para "atacá-lo"

## 🔤 Sintaxe Clojure

### Estrutura Uniforme
Clojure tem estrutura muito simples e uniforme. Tudo segue o mesmo padrão:

```clojure
(operador operando1 operando2 ... operandoN)
```

### Formas (Forms)
Clojure reconhece dois tipos de estruturas:
1. **Literais**: Representações diretas de dados
2. **Operações**: Fazem coisas com os dados

```clojure
;; Literais
1
"uma string"
["um" "vetor" "de" "strings"]

;; Operações  
(+ 1 2 3)        ; => 6
(str "Olá" " " "mundo")  ; => "Olá mundo"
```

### Controle de Fluxo

#### if - Condicional
```clojure
(if condicao-booleana
  forma-then
  forma-else-opcional)

;; Exemplos
(if true
  "Verdadeiro!"
  "Falso!")
; => "Verdadeiro!"

(if false
  "Nunca executado")
; => nil
```

#### do - Múltiplas Formas
```clojure
(if true
  (do (println "Sucesso!")
      "Resultado final")
  (do (println "Falha!")
      "Outro resultado"))
; => Sucesso!
; => "Resultado final"
```

#### when - If sem Else
```clojure
(when true
  (println "Executando...")
  "valor de retorno")
; => Executando...
; => "valor de retorno"
```

### Truthiness e Falsiness
- **Falsy**: `nil` e `false`
- **Truthy**: Todo o resto (incluindo 0, "", [])

```clojure
(if "qualquer string"
  "strings são truthy")
; => "strings são truthy"

(if nil
  "nunca executado"
  "nil é falsy")
; => "nil é falsy"
```

### Operadores Booleanos
```clojure
;; or - retorna primeiro valor truthy ou último valor
(or false nil :primeiro-truthy :nunca-alcançado)
; => :primeiro-truthy

(or false nil)
; => nil

;; and - retorna primeiro falsy ou último truthy
(and :primeiro :segundo :terceiro)
; => :terceiro

(and :primeiro nil :nunca-alcançado)
; => nil
```

### Naming com def
```clojure
(def lista-herois
  ["Superman" "Batman" "Wonder Woman"])

lista-herois
; => ["Superman" "Batman" "Wonder Woman"]
```

**⚠️ Importante**: Use `def` como constantes, não como variáveis mutáveis!

## 📊 Estruturas de Dados

Todas as estruturas em Clojure são **imutáveis** - não podem ser modificadas no local.

### Numbers
```clojure
93        ; inteiro
1.2       ; float  
1/5       ; ratio (fração)
```

### Strings
```clojure
"Uma string"
"String com \\"aspas\\" escapadas"

;; Concatenação
(def nome "João")
(str "Olá, " nome "!")
; => "Olá, João!"
```

### Maps (Mapas)
Associam chaves a valores (como dicionários/objetos):

```clojure
;; Map vazio
{}

;; Map com keywords
{:nome "Ana"
 :idade 25
 :cidade "São Paulo"}

;; Map com strings
{"chave-string" "valor"}

;; Maps aninhados
{:pessoa {:nome "Carlos" :sobrenome "Silva"}}
```

**Acessando valores:**
```clojure
(def pessoa {:nome "Ana" :idade 25})

(get pessoa :nome)           ; => "Ana"
(get pessoa :altura "N/A")  ; => "N/A" (valor padrão)

;; Map como função
(pessoa :nome)               ; => "Ana"

;; Keyword como função  
(:nome pessoa)               ; => "Ana"
```

**Maps aninhados:**
```clojure
(def dados {:pessoa {:nome "João" :idade 30}})
(get-in dados [:pessoa :nome])  ; => "João"
```

### Keywords
Identificadores especiais que começam com `:`:

```clojure
:nome
:idade  
:categoria
:_qualquer-coisa

;; Como função para lookup
(:nome {:nome "Ana" :idade 25})  ; => "Ana"
(:altura {:nome "Ana"} "N/A")    ; => "N/A"
```

### Vectors (Vetores)
Coleções ordenadas e indexadas (como arrays):

```clojure
[1 2 3 4]
["misturado" 42 :keyword {:mapa "valor"}]

;; Acessando por índice
(get ["a" "b" "c"] 0)         ; => "a"
(get ["a" "b" "c"] 1)         ; => "b"

;; Criando vetores
(vector "um" "dois" "três")    ; => ["um" "dois" "três"]

;; Adicionando elementos (ao final)
(conj [1 2 3] 4)              ; => [1 2 3 4]
```

### Lists (Listas)
Coleções ordenadas otimizadas para acesso sequencial:

```clojure
'(1 2 3 4)                    ; Literal com quote

;; Acessando por posição
(nth '(:a :b :c) 0)           ; => :a
(nth '(:a :b :c) 2)           ; => :c

;; Criando listas
(list 1 "dois" {:tres 4})    ; => (1 "dois" {:tres 4})

;; Adicionando elementos (ao início)
(conj '(1 2 3) 4)             ; => (4 1 2 3)
```

**Vector vs List:**
- **Vector**: Use quando precisa de acesso rápido por índice
- **List**: Use quando vai adicionar no início ou escrevendo macros

### Sets (Conjuntos)
Coleções de valores únicos:

```clojure
#{"valor1" "valor2" :keyword}

;; Criando sets
(hash-set 1 1 2 2 3)          ; => #{1 2 3}
(set [1 1 2 2 3])             ; => #{1 2 3}

;; Adicionando (duplicatas ignoradas)
(conj #{:a :b} :b)            ; => #{:a :b}

;; Verificando pertencimento
(contains? #{:a :b} :a)       ; => true
(:a #{:a :b})                 ; => :a
(get #{:a :b} :a)             ; => :a
```

### Filosofia da Simplicidade
> "É melhor ter 100 funções operando em uma estrutura de dados do que 10 funções em 10 estruturas."
> — Alan Perlis

## 🔧 Funções

### Chamando Funções
```clojure
(+ 1 2 3)                     ; => 6
(* 2 3 4)                     ; => 24
(str "a" "b" "c")              ; => "abc"
```

**Expressões como operadores:**
```clojure
((or + -) 1 2 3)              ; => 6
((first [+ *]) 10 5)          ; => 15
```

**Funções de alta ordem:**
```clojure
(map inc [0 1 2 3])           ; => (1 2 3 4)
```

### Definindo Funções
```clojure
(defn nome-da-funcao
  "Docstring opcional"
  [parametros]
  corpo-da-funcao)

;; Exemplo
(defn saudacao
  "Cria uma saudação personalizada"
  [nome]
  (str "Olá, " nome "!"))

(saudacao "Maria")            ; => "Olá, Maria!"
```

#### Aridade Múltipla
```clojure
(defn multi-arity
  ([x] (multi-arity x 0))         ; 1 parâmetro chama versão de 2
  ([x y] (+ x y)))                ; 2 parâmetros

(multi-arity 5)               ; => 5  
(multi-arity 5 3)             ; => 8
```

#### Parâmetros Rest
```clojure
(defn var-args
  [primeiro & resto]
  (str "Primeiro: " primeiro 
       ", Resto: " resto))

(var-args "a" "b" "c" "d")
; => "Primeiro: a, Resto: (\\"b\\" \\"c\\" \\"d\\")"
```

#### Destructuring
**Vetores/Listas:**
```clojure
(defn meu-primeiro
  [[primeiro-item]]             ; Destructuring do primeiro elemento
  primeiro-item)

(meu-primeiro ["a" "b" "c"])  ; => "a"

;; Com rest parameters
(defn escolhedor  
  [[primeira segunda & outras]]
  (println "Primeira:" primeira)
  (println "Segunda:" segunda)
  (println "Outras:" outras))
```

**Maps:**
```clojure
(defn localizar-tesouro
  [{:keys [lat lng]}]           ; Extrai :lat e :lng
  (println "Latitude:" lat)
  (println "Longitude:" lng))

(localizar-tesouro {:lat 28.22 :lng 81.33})

;; Com :as para manter referência original
(defn processar-coordenadas
  [{:keys [lat lng] :as coords}]
  (println "Processando:" coords)
  (+ lat lng))
```

### Funções Anônimas
**Forma fn:**
```clojure
(fn [x] (* x 2))              ; Função anônima

(map (fn [nome] (str "Oi, " nome))
     ["Ana" "João"])
; => ("Oi, Ana" "Oi, João")

;; Com nome local
(def dobrar (fn [x] (* x 2)))
```

**Forma #():**
```clojure
#(* % 2)                      ; Forma compacta

(map #(str "Oi, " %) ["Ana" "João"])
; => ("Oi, Ana" "Oi, João")

;; Múltiplos parâmetros
#(str %1 " e " %2)           ; %1, %2, %3...

;; Rest parameters
#(println %&)                 ; %& = rest args
```

### Retornando Funções (Closures)
```clojure
(defn criar-incrementador
  [n]
  #(+ % n))                   ; Closure - acessa 'n'

(def inc3 (criar-incrementador 3))
(inc3 7)                      ; => 10
```

## 🔗 Ferramentas Auxiliares

### let - Binding Local
```clojure
(let [x 1
      y 2]
  (+ x y))                    ; => 3

;; Com destructuring
(let [[primeiro & resto] [1 2 3 4]]
  (str "Primeiro: " primeiro 
       ", Resto: " resto))
```

### loop - Recursão
```clojure
(loop [contador 0]
  (println "Contador:" contador)
  (if (< contador 3)
    (recur (inc contador))    ; Chama loop novamente
    "Fim!"))
```

### Expressões Regulares
```clojure
#"padrão"                   ; Literal regex

(re-find #"^left-" "left-eye")   ; => "left-"
(re-find #"^left-" "right-eye")  ; => nil

;; Substituição
(clojure.string/replace "left-hand" #"^left-" "right-")
; => "right-hand"
```

### reduce - Padrão Fundamental
```clojure
(reduce + [1 2 3 4])          ; => 10
; Equivale a: (+ (+ (+ 1 2) 3) 4)

(reduce + 10 [1 2 3 4])       ; => 20 (com valor inicial)

;; Construindo coleções
(reduce conj [] [:a :b :c])   ; => [:a :b :c]
```

## 🗡️ Projeto Prático: Atacando Hobbits

### Modelo do Hobbit
```clojure
(def partes-corpo-assimetricas
  [{:nome "cabeça" :tamanho 3}
   {:nome "olho-esquerdo" :tamanho 1}
   {:nome "orelha-esquerda" :tamanho 1}
   {:nome "ombro-esquerdo" :tamanho 3}
   ;; ... mais partes esquerdas
   ])
```

### Função para Criar Parte Correspondente
```clojure
(defn parte-correspondente
  [parte]
  {:nome (clojure.string/replace (:nome parte) #"^esquerdo-" "direito-")
   :tamanho (:tamanho parte)})
```

### Simetrizador com reduce
```clojure
(defn simetrizar-partes-corpo
  [partes-assimetricas]
  (reduce (fn [partes-finais parte]
            (into partes-finais 
                  (set [parte (parte-correspondente parte)])))
          []
          partes-assimetricas))
```

### Função de Ataque
```clojure
(defn atingir
  [partes-assimetricas]
  (let [partes-simetricas (simetrizar-partes-corpo partes-assimetricas)
        soma-tamanhos (reduce + (map :tamanho partes-simetricas))
        alvo (rand soma-tamanhos)]
    (loop [[parte & restantes] partes-simetricas
           tamanho-acumulado (:tamanho parte)]
      (if (> tamanho-acumulado alvo)
        parte
        (recur restantes 
               (+ tamanho-acumulado (:tamanho (first restantes))))))))

;; Testando
(atingir partes-corpo-assimetricas)
; => {:nome "ombro-direito", :tamanho 3}
```

## 📚 Conceitos-Chave Resumidos

### 🎯 Sintaxe
- Estrutura uniforme: `(operador operandos...)`
- Notação prefixa em tudo
- Formas = código válido

### 🏗️ Estruturas de Dados
- **Imutáveis** por padrão
- **Numbers, Strings**: Tipos básicos
- **Maps**: `{:chave valor}` - associações
- **Keywords**: `:palavra` - identificadores
- **Vectors**: `[1 2 3]` - sequências indexadas
- **Lists**: `'(1 2 3)` - sequências linkadas
- **Sets**: `#{1 2 3}` - valores únicos

### ⚡ Funções
- **Cidadãos de primeira classe** - podem ser valores
- **Higher-order**: recebem/retornam outras funções  
- **Aridade múltipla**: diferentes números de parâmetros
- **Destructuring**: decomposição automática de estruturas
- **Anônimas**: `fn` e `#()` 
- **Closures**: capturam ambiente

### 🔧 Ferramentas
- **let**: binding local de nomes
- **loop/recur**: recursão otimizada  
- **reduce**: padrão processo-e-acumule
- **Regex**: `#"padrão"` matching

### 🎨 Filosofia
- **Simplicidade**: poucas estruturas, muitas funções
- **Composição**: funções pequenas que se combinam
- **Imutabilidade**: dados não mudam
- **Expressividade**: código como dados

## 🚀 Próximos Passos

1. **Pratique no REPL** - digite todos os exemplos
2. **Experimente variações** - mude os exemplos
3. **Faça os exercícios** do final do capítulo
4. **Use o Cheat Sheet**: [clojure.org/api/cheatsheet](http://clojure.org/api/cheatsheet)
5. **Desafios**: [4Clojure.com](http://www.4clojure.com/) e [Project Euler](http://www.projecteuler.net/)

## 💻 Exercícios Recomendados

1. Use `str`, `vector`, `list`, `hash-map`, `hash-set`
2. Escreva função que adiciona 100 a um número
3. Implemente `dec-maker` (como `inc-maker` mas subtraindo)
4. Escreva `mapset` (como `map` mas retorna set)
5. Crie simetrizador para alienígenas com 5 membros
6. Generalize o simetrizador para N membros

---

> **Próximo**: Capítulo 4 - Core Functions in Depth

> **Lembre-se**: Este capítulo é a base de tudo - pratique muito!