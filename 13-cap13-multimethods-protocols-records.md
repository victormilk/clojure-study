# Capítulo 13: Creating and Extending Abstractions with Multimethods, Protocols, and Records

## 🎯 Conceito Central: Abstrações e Polimorfismo

**Abstração** = conjunto de operações
**Data types** = implementações de abstrações
**Polimorfismo** = um nome de operação → múltiplos algoritmos

### Por Que Abstrações São Poderosas?
- **Poder cognitivo**: Você pensa em conceitos, não implementações
- **Produtividade**: Se sabe que é "seq", conhece centenas de funções
- **Biblioteca unificada**: Mesma interface para diferentes tipos

**Analogia**: "Nariz de palhaço" é mais eficiente mentalmente que "enfeite vermelho espremível que faz barulho"

## 🏗️ Multimethods: Polimorfismo Flexível

### Conceito Base
```clojure
;; Dispatching function decide qual method usar
;; Como maître do restaurante: pergunta → direciona à mesa certa

(defmulti nome-multimethod dispatching-function)
(defmethod nome-multimethod dispatch-value [args] body)
```

### Exemplo: Were-Creatures
```clojure
(ns were-creatures)

;; 1. Define multimethod com dispatching function
(defmulti full-moon-behavior (fn [were-creature] (:were-type were-creature)))

;; 2. Define methods para cada dispatch value
(defmethod full-moon-behavior :wolf
  [were-creature]
  (str (:name were-creature) " will howl and murder"))

(defmethod full-moon-behavior :simmons
  [were-creature]
  (str (:name were-creature) " will encourage people and sweat to the oldies"))

;; 3. Uso
(full-moon-behavior {:were-type :wolf :name "Rachel"})
; => "Rachel will howl and murder"

(full-moon-behavior {:were-type :simmons :name "Andy"})
; => "Andy will encourage people and sweat to the oldies"
```

### Sequência de Dispatch
```
1. (full-moon-behavior {:were-type :wolf :name "Rachel"}) → chamada
2. Dispatching function executa → retorna :wolf (dispatching value)
3. Clojure compara :wolf com dispatch values dos methods (:wolf, :simmons)
4. Match encontrado → executa method correspondente
```

### Casos Especiais
```clojure
;; Method para nil
(defmethod full-moon-behavior nil
  [were-creature]
  (str (:name were-creature) " will stay at home and eat ice cream"))

;; Method default
(defmethod full-moon-behavior :default
  [were-creature]
  (str (:name were-creature) " will stay up all night fantasy footballing"))

(full-moon-behavior {:were-type nil :name "Martin"})
; => "Martin will stay at home and eat ice cream"

(full-moon-behavior {:were-type :office-worker :name "Jimmy"})
; => "Jimmy will stay up all night fantasy footballing"
```

### Extensibilidade
```clojure
;; Outros namespaces podem adicionar methods!
(ns random-namespace
  (:require [were-creatures]))

(defmethod were-creatures/full-moon-behavior :bill-murray
  [were-creature]
  (str (:name were-creature) " will be the most likeable celebrity"))

(were-creatures/full-moon-behavior {:name "Laura" :were-type :bill-murray})
; => "Laura will be the most likeable celebrity"
```

### Dispatch Múltiplo
```clojure
;; Dispatching function pode usar qualquer lógica
(defmulti types (fn [x y] [(class x) (class y)]))

(defmethod types [java.lang.String java.lang.String]
  [x y]
  "Two strings!")

(types "String 1" "String 2")
; => "Two strings!"
```

## 📋 Protocols: Polimorfismo Otimizado por Tipo

### Características
- **93.58%** das vezes você quer dispatch por tipo
- **Protocols** são otimizados para isso
- **Mais eficientes** que multimethods
- **Coleção** de operações polimórficas

### Definindo Protocol
```clojure
(ns data-psychology)

(defprotocol Psychodynamics
  "Plumb the inner depths of your data types"
  (thoughts [x] "The data type's innermost thoughts")
  (feelings-about [x] [x y] "Feelings about self or other"))
```

### Implementando Protocol
```clojure
;; extend-type: Implementa protocol para um tipo
(extend-type java.lang.String
  Psychodynamics
  (thoughts [x] 
    (str x " thinks, 'Truly, the character defines the data type'"))
  (feelings-about
    ([x] (str x " is longing for a simpler way of life"))
    ([x y] (str x " is envious of " y "'s simpler way of life"))))

;; Uso
(thoughts "blorb")
; => "blorb thinks, 'Truly, the character defines the data type'"

(feelings-about "schmorb")
; => "schmorb is longing for a simpler way of life"

(feelings-about "schmorb" 2)
; => "schmorb is envious of 2's simpler way of life"
```

### Default Implementation
```clojure
;; java.lang.Object = todos os tipos descendem dele
(extend-type java.lang.Object
  Psychodynamics
  (thoughts [x] "Maybe the Internet is just a vector for toxoplasmosis")
  (feelings-about
    ([x] "meh")
    ([x y] (str "meh about " y))))

(thoughts 3)
; => "Maybe the Internet is just a vector for toxoplasmosis"
```

### extend-protocol: Múltiplos Tipos
```clojure
(extend-protocol Psychodynamics
  java.lang.String
  (thoughts [x] "Truly, the character defines the data type")
  (feelings-about
    ([x] "longing for a simpler way of life")
    ([x y] (str "envious of " y "'s simpler way of life")))

  java.lang.Object
  (thoughts [x] "Maybe the Internet is just a vector for toxoplasmosis")
  (feelings-about
    ([x] "meh")
    ([x y] (str "meh about " y))))
```

### Methods Belong to Namespace
```
Protocol methods "pertencem" ao namespace onde foram definidos:
- data-psychology/thoughts
- data-psychology/feelings-about

OOP: methods pertencem ao tipo
Clojure: methods pertencem ao namespace (primazia das abstrações)
```

## 📦 Records: Custom Data Types

### O Que São Records?
- **Custom maplike data types**
- **Especificam fields** (chaves obrigatórias)
- **Immutable** como maps
- **Implementam protocols**
- **Performance** superior aos maps

### Definindo Record
```clojure
(ns were-records)
(defrecord WereWolf [name title])
```

### Criando Instâncias
```clojure
;; 1. Java interop style
(WereWolf. "David" "London Tourist")
; => #were_records.WereWolf{:name "David", :title "London Tourist"}

;; 2. Factory function (argumentos posicionais)
(->WereWolf "Jacob" "Lead Shirt Discarder")
; => #were_records.WereWolf{:name "Jacob", :title "Lead Shirt Discarder"}

;; 3. Map factory function
(map->WereWolf {:name "Lucian" :title "CEO of Melodrama"})
; => #were_records.WereWolf{:name "Lucian", :title "CEO of Melodrama"}
```

### Importando Records
```clojure
;; ATENÇÃO: dashes → underscores no namespace
(ns monster-mash
  (:import [were_records WereWolf]))

(WereWolf. "David" "London Tourist")
```

### Acessando Values
```clojure
(def jacob (->WereWolf "Jacob" "Lead Shirt Discarder"))

;; Java interop
(.name jacob)           ; => "Jacob"

;; Como map
(:name jacob)           ; => "Jacob"
(get jacob :name)       ; => "Jacob"
```

### Equality Testing
```clojure
;; Igualdade: mesmo tipo + mesmos fields
(= jacob (->WereWolf "Jacob" "Lead Shirt Discarder"))  ; => true
(= jacob (WereWolf. "David" "London Tourist"))         ; => false
(= jacob {:name "Jacob" :title "Lead Shirt Discarder"}) ; => false (tipos diferentes)
```

### Map Functions
```clojure
;; Qualquer função de map funciona
(assoc jacob :title "Lead Third Wheel")
; => #were_records.WereWolf{:name "Jacob", :title "Lead Third Wheel"}

;; MAS: dissoc field → vira plain map (não mais record)
(dissoc jacob :title)
; => {:name "Jacob"}  ;; ← Perdeu o tipo WereWolf!
```

### ⚠️ Consequências do dissoc
1. **Performance**: Map access mais lento que record access
2. **Protocol methods**: Não funcionam mais no resultado

### Implementando Protocols em Records
```clojure
;; Define protocol
(defprotocol WereCreature
  (full-moon-behavior [x]))

;; Record implementa protocol na definição
(defrecord WereWolf [name title]
  WereCreature
  (full-moon-behavior [x]
    (str name " will howl and murder")))  ; ← acesso direto aos fields

(full-moon-behavior (map->WereWolf {:name "Lucian" :title "CEO of Melodrama"}))
; => "Lucian will howl and murder"
```

### Quando Usar Records vs Maps?

**Use Records quando**:
- **Same fields repetidos**: Representa conceito do domínio
- **Performance**: Access mais rápido
- **Protocols**: Precisa implementar protocols

**Use Maps quando**:
- **Flexibilidade**: Estrutura varia muito
- **Ad-hoc data**: Dados temporários/transitórios
- **Simplicidade**: Não justifica complexidade de record

## 📚 Conceitos-Chave Resumidos

### 🎭 Polimorfismo Comparison
| Tipo | Dispatch | Performance | Use Case |
|------|----------|-------------|----------|
| **Multimethod** | Arbitrary function | Slower | Complex dispatch logic |
| **Protocol** | Type of first arg | Faster | Type-based dispatch |

### 🏗️ Multimethods
- **defmulti**: Define dispatching function
- **defmethod**: Implementa method para dispatch value
- **Extensible**: Qualquer namespace pode adicionar methods
- **Flexible**: Dispatch por qualquer critério
- **:default**: Method padrão
- **Multiple dispatch**: Pode usar múltiplos argumentos

### 📋 Protocols
- **defprotocol**: Define conjunto de methods
- **extend-type**: Implementa para um tipo
- **extend-protocol**: Implementa para múltiplos tipos
- **Type dispatch only**: Sempre pelo tipo do 1º argumento
- **Namespace ownership**: Methods pertencem ao namespace
- **Performance**: Otimizado para dispatch por tipo

### 📦 Records
- **defrecord**: Define custom data type
- **Fields**: Chaves obrigatórias especificadas
- **Factory functions**: `->RecordName`, `map->RecordName`
- **Map compatible**: Todas funções de map funcionam
- **Protocol implementation**: Na definição ou via extend
- **dissoc warning**: Field removal → plain map

## 💡 Melhores Práticas

### ✅ Do
- **Multimethods** para dispatch complexo ou customizado
- **Protocols** para dispatch por tipo (93.58% dos casos)
- **Records** para dados estruturados do domínio
- **Import records** com underscores (were_records)
- **Default methods** (:default para multimethods, Object para protocols)
- **Implement all protocol methods** (obrigatório)

### ❌ Don't
- **Avoid dissoc on records** se precisar manter tipo
- **Não misture**: protocol methods em namespaces diferentes
- **Protocol rest args**: Não permitido `[x & others]`
- **Skip implementations**: Protocol exige todos os methods

### 🎯 Quando Usar O Quê
- **Maps**: Dados flexíveis, ad-hoc, simples
- **Records**: Dados estruturados do domínio + protocols
- **Multimethods**: Dispatch complexo (RPG: spell school + specialization)
- **Protocols**: Dispatch simples por tipo (seq, coll, etc.)

## 🚀 Padrões Avançados

### Hierarchical Dispatch
```clojure
;; Multimethods suportam hierarquias customizadas
;; http://clojure.org/multimethods/ para detalhes
```

### Advanced Constructs
- **deftype**: Lower-level que records
- **reify**: Anonymous implementations  
- **proxy**: Java class/interface implementation

---

> **Próximo**: Apêndice A - Building and Developing with Leiningen

> **Lição Principal**: Abstrações = poder conceitual. Multimethods = flexibilidade. Protocols = performance. Records = dados estruturados!