# Capítulo 4: Core Functions in Depth

## 🎯 Objetivos do Capítulo
- Entender programação para abstrações
- Dominar as abstrações de sequence e collection
- Aprender lazy sequences e sua eficiência
- Usar funções de alta ordem (apply, partial, complement)
- Construir um programa de análise de vampiros

## 🔄 Programação para Abstrações

### Conceito Central
Clojure define `map`, `reduce` e outras funções em termos de **abstrações**, não estruturas de dados específicas. Se uma estrutura responde às operações centrais (`first`, `rest`, `cons`), ela funciona com todas as funções seq.

### Exemplo Comparativo
```clojure
;; Em Emacs Lisp (sem abstrações)
(mapcar func list)      ; Para listas
(maphash func hashmap)  ; Para hash maps

;; Em Clojure (com abstrações)
(map func any-seq-structure)  ; Para qualquer sequência
```

### Tratando Todas as Estruturas como Sequências
```clojure
(defn titleize [topic] (str topic \" for the Brave and True\"))

;; Funciona com vetores
(map titleize [\"Hamsters\" \"Ragnarok\"])
; => (\"Hamsters for the Brave and True\" \"Ragnarok for the Brave and True\")

;; Funciona com listas
(map titleize '(\"Empathy\" \"Decorating\"))
; => (\"Empathy for the Brave and True\" \"Decorating for the Brave and True\")

;; Funciona com sets
(map titleize #{\"Elbows\" \"Soap Carving\"})
; => (\"Elbows for the Brave and True\" \"Soap Carving for the Brave and True\")

;; Funciona com maps
(map #(titleize (second %)) {:uncomfortable-thing \"Winking\"})
; => (\"Winking for the Brave and True\")
```

### As Três Funções Centrais: first, rest, cons

**JavaScript: Implementação de Lista Linkada**
```javascript
// Estrutura de nó
var node1 = { value: \"first\", next: node2 };
var node2 = { value: \"middle\", next: node3 };
var node3 = { value: \"last\", next: null };

// Funções centrais
var first = function(node) { return node.value; };
var rest = function(node) { return node.next; };
var cons = function(newValue, node) { 
  return { value: newValue, next: node }; 
};

// map implementado com first, rest, cons
var map = function (list, transform) {
  if (list === null) {
    return null;
  } else {
    return cons(transform(first(list)), map(rest(list), transform));
  }
}
```

### Abstração por Indireção
Clojure usa **polimorfismo** e **conversão de tipos** para fazer `first`, `rest` e `cons` funcionarem com diferentes estruturas.

**A função `seq`:**
```clojure
(seq '(1 2 3))      ; => (1 2 3)
(seq [1 2 3])       ; => (1 2 3)  
(seq #{1 2 3})      ; => (1 2 3)
(seq {:a 1 :b 2})   ; => ([:a 1] [:b 2])

;; Converter de volta para map
(into {} (seq {:a 1 :b 2 :c 3}))
; => {:a 1, :c 3, :b 2}
```

## 🔧 Exemplos de Funções Seq

### map - Versátil e Poderosa
**Múltiplas coleções:**
```clojure
(map str [\"a\" \"b\" \"c\"] [\"A\" \"B\" \"C\"])
; => (\"aA\" \"bB\" \"cC\")

;; Unificando dados de dieta de vampiro
(def human-consumption [8.1 7.3 6.6 5.0])
(def critter-consumption [0.0 0.2 0.3 1.1])

(defn unify-diet-data [human critter]
  {:human human :critter critter})

(map unify-diet-data human-consumption critter-consumption)
; => ({:human 8.1, :critter 0.0}
      {:human 7.3, :critter 0.2}
      {:human 6.6, :critter 0.3}
      {:human 5.0, :critter 1.1})
```

**Coleção de funções:**
```clojure
(def sum #(reduce + %))
(def avg #(/ (sum %) (count %)))

(defn stats [numbers]
  (map #(% numbers) [sum count avg]))

(stats [3 4 10])    ; => (17 3 17/3)
```

**Keywords como funções:**
```clojure
(def identities
  [{:alias \"Batman\" :real \"Bruce Wayne\"}
   {:alias \"Spider-Man\" :real \"Peter Parker\"}])

(map :real identities)
; => (\"Bruce Wayne\" \"Peter Parker\")
```

### reduce - Mais Flexível que Parece
**Transformando valores de map:**
```clojure
(reduce (fn [new-map [key val]]
          (assoc new-map key (inc val)))
        {}
        {:max 30 :min 10})
; => {:max 31, :min 11}
```

**Filtrando por valor:**
```clojure
(reduce (fn [new-map [key val]]
          (if (> val 4)
            (assoc new-map key val)
            new-map))
        {}
        {:human 4.1 :critter 3.9})
; => {:human 4.1}
```

### take, drop, take-while, drop-while
```clojure
(take 3 [1 2 3 4 5])        ; => (1 2 3)
(drop 3 [1 2 3 4 5])        ; => (4 5)

;; Diário alimentar de vampiro
(def food-journal
  [{:month 1 :day 1 :human 5.3}
   {:month 1 :day 2 :human 5.1}
   {:month 2 :day 1 :human 4.9}])

;; Dados de janeiro e fevereiro
(take-while #(< (:month %) 3) food-journal)

;; Dados de março em diante
(drop-while #(< (:month %) 3) food-journal)

;; Apenas fevereiro e março
(take-while #(< (:month %) 4)
            (drop-while #(< (:month %) 2) food-journal))
```

### filter e some
```clojure
;; Entradas com consumo humano < 5 litros
(filter #(< (:human %) 5) food-journal)

;; Existe entrada com critter > 3?
(some #(> (:critter %) 3) food-journal)  ; => true

;; Retornar a entrada que satisfaz
(some #(and (> (:critter %) 3) %) food-journal)
; => {:month 3 :day 1 :human 4.2 :critter 3.3}
```

### sort e sort-by
```clojure
(sort [3 1 2])              ; => (1 2 3)

;; Ordenar por função
(sort-by count [\"aaa\" \"c\" \"bb\"])
; => (\"c\" \"bb\" \"aaa\")
```

### concat
```clojure
(concat [1 2] [3 4])        ; => (1 2 3 4)
```

## ⚡ Lazy Sequences

### Eficiência Demonstrada
```clojure
;; Banco de vampiros
(def vampire-database
  {0 {:makes-blood-puns? false, :has-pulse? true  :name \"McFishwich\"}
   1 {:makes-blood-puns? false, :has-pulse? true  :name \"McMackson\"}
   2 {:makes-blood-puns? true,  :has-pulse? false :name \"Damon Salvatore\"}})

(defn vampire-related-details [social-security-number]
  (Thread/sleep 1000)  ; Simula 1 segundo de busca
  (get vampire-database social-security-number))

(defn vampire? [record]
  (and (:makes-blood-puns? record)
       (not (:has-pulse? record))
       record))

;; map retorna lazy seq - instantâneo!
(time (def mapped-details (map vampire-related-details (range 0 1000000))))
; => \"Elapsed time: 0.049 msecs\"

;; Acessar primeiro elemento realiza chunk (32 elementos)
(time (first mapped-details))
; => \"Elapsed time: 32030.767 msecs\"

;; Segunda chamada é instantânea (já realizada)
(time (first mapped-details))  
; => \"Elapsed time: 0.022 msecs\"
```

### Sequências Infinitas
```clojure
;; Batman theme
(concat (take 8 (repeat \"na\")) [\"Batman!\"])
; => (\"na\" \"na\" \"na\" \"na\" \"na\" \"na\" \"na\" \"na\" \"Batman!\")

;; Números aleatórios
(take 3 (repeatedly (fn [] (rand-int 10))))
; => (1 4 0)

;; Números pares infinitos
(defn even-numbers
  ([] (even-numbers 0))
  ([n] (cons n (lazy-seq (even-numbers (+ n 2))))))

(take 10 (even-numbers))
; => (0 2 4 6 8 10 12 14 16 18)
```

## 📦 Abstração de Collection

### into - Conversão Entre Tipos
```clojure
;; map retorna seq, into converte de volta
(into {} (map identity {:sunlight-reaction \"Glitter!\"}))
; => {:sunlight-reaction \"Glitter!\"}

(into [] (map identity [:garlic :sesame-oil :fried-eggs]))
; => [:garlic :sesame-oil :fried-eggs]

;; Adicionar elementos
(into {:favorite-emotion \"gloomy\"} 
      [[:sunlight-reaction \"Glitter!\"]])
; => {:favorite-emotion \"gloomy\" :sunlight-reaction \"Glitter!\"}
```

### conj - Adicionar Elementos
```clojure
(conj [0] 1)           ; => [0 1]
(conj [0] 1 2 3 4)     ; => [0 1 2 3 4]

;; Diferença: conj adiciona elementos individuais
(conj [0] [1])         ; => [0 [1]]
(into [0] [1])         ; => [0 1]

;; Definindo conj em termos de into
(defn my-conj [target & additions]
  (into target additions))
```

## 🔧 Function Functions

### apply - \"Explode\" Coleções
```clojure
;; Problema: max não aceita vetor
(max [0 1 2])          ; => [0 1 2] (errado!)

;; Solução: apply explode o vetor
(apply max [0 1 2])    ; => 2

;; Definindo into com apply
(defn my-into [target additions]
  (apply conj target additions))
```

### partial - Aplicação Parcial
```clojure
;; Criar função especializada
(def add10 (partial + 10))
(add10 3)              ; => 13

(def add-missing-elements
  (partial conj [\"water\" \"earth\" \"air\"]))
(add-missing-elements \"unobtainium\" \"adamantium\")
; => [\"water\" \"earth\" \"air\" \"unobtainium\" \"adamantium\"]

;; Logger especializado
(defn lousy-logger [log-level message]
  (condp = log-level
    :warn (clojure.string/lower-case message)
    :emergency (clojure.string/upper-case message)))

(def warn (partial lousy-logger :warn))
(warn \"Red light ahead\")
; => \"red light ahead\"

;; Implementação de partial
(defn my-partial [partialized-fn & args]
  (fn [& more-args]
    (apply partialized-fn (into args more-args))))
```

### complement - Negação de Função
```clojure
;; Problema: queremos NOT vampire?
(defn identify-humans [social-security-numbers]
  (filter #(not (vampire? %))
          (map vampire-related-details social-security-numbers)))

;; Solução: complement
(def not-vampire? (complement vampire?))
(defn identify-humans [social-security-numbers]
  (filter not-vampire?
          (map vampire-related-details social-security-numbers)))

;; Implementação de complement
(defn my-complement [fun]
  (fn [& args]
    (not (apply fun args))))
```

## 🧛 Programa de Análise de Vampiros

### Estrutura do Projeto
```bash
lein new app fwpd
```

**suspects.csv:**
```
Edward Cullen,10
Bella Swan,0
Charlie Swan,0
Jacob Black,3
Carlisle Cullen,6
```

### Código Principal
```clojure
(ns fwpd.core)
(def filename \"suspects.csv\")

;; Configuração de conversões
(def vamp-keys [:name :glitter-index])
(defn str->int [str] (Integer. str))
(def conversions {:name identity :glitter-index str->int})
(defn convert [vamp-key value] ((get conversions vamp-key) value))

;; Parser CSV
(defn parse [string]
  (map #(clojure.string/split % #\",\")
       (clojure.string/split string #\"\\n\")))

;; Converter para mapas
(defn mapify [rows]
  (map (fn [unmapped-row]
         (reduce (fn [row-map [vamp-key value]]
                   (assoc row-map vamp-key (convert vamp-key value)))
                 {}
                 (map vector vamp-keys unmapped-row)))
       rows))

;; Filtrar por brilho
(defn glitter-filter [minimum-glitter records]
  (filter #(>= (:glitter-index %) minimum-glitter) records))

;; Usar tudo junto
(glitter-filter 3 (mapify (parse (slurp filename))))
; => ({:name \"Edward Cullen\", :glitter-index 10}
      {:name \"Jacob Black\", :glitter-index 3}
      {:name \"Carlisle Cullen\", :glitter-index 6})
```

## 📚 Conceitos-Chave Resumidos

### 🎯 Programação para Abstrações
- **Foco em operações**, não implementações
- **Sequence abstraction**: `first`, `rest`, `cons`
- **Collection abstraction**: operações no conjunto todo
- **Reutilização massiva**: mesmas funções, diferentes estruturas

### ⚡ Lazy Sequences  
- **Computação adiada** até necessária
- **Chunking**: realiza 32 elementos por vez
- **Eficiência**: evita processamento desnecessário
- **Sequências infinitas** são possíveis

### 🔧 Funções de Alta Ordem
- **apply**: explode coleções em argumentos
- **partial**: aplicação parcial de funções
- **complement**: negação de predicados
- **Composabilidade**: funções que criam funções

### 📊 Padrões Funcionais
- **Transformação**: `map`, `filter`, `reduce`
- **Particionamento**: `take`, `drop`, `take-while`, `drop-while`
- **Agregação**: `reduce`, `some`, `every?`
- **Conversão**: `into`, `conj`, seq functions

### 🏗️ Estrutura de Dados
- **Imutabilidade**: estruturas não mudam
- **Polimorfismo**: mesma interface, diferentes tipos
- **Indireção**: abstrações escondem implementação
- **Eficiência**: structural sharing

## 💡 Principais Lições

### ✅ **Vantagens das Abstrações**
- Código mais reutilizável
- Menos funções específicas por tipo
- Raciocínio mais simples
- Composição poderosa

### ✅ **Poder das Lazy Sequences**
- Performance melhor em grandes datasets
- Sequências infinitas
- Processamento sob demanda
- Chunking automático

### ✅ **Flexibilidade de Higher-Order Functions**
- Adaptação de funções existentes
- Criação de funções especializadas
- Composição elegante
- Reutilização de padrões

## 🚀 Exercícios Sugeridos

1. **Transformar resultado em lista de nomes**
2. **Escrever função `append` para adicionar suspeitos**  
3. **Criar função `validate` para validar entrada**
4. **Converter lista de mapas de volta para CSV**

---

> **Próximo**: Capítulo 5 - Functional Programming

> **Dica**: Pratique com sequence functions - elas são o coração do Clojure!