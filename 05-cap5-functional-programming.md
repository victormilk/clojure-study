# Capítulo 5: Functional Programming

## 🎯 Objetivos do Capítulo
- Entender o que são funções puras e por que são úteis
- Trabalhar com estruturas de dados imutáveis
- Aprender como separar dados e funções dá mais poder
- Programar para um pequeno conjunto de abstrações
- Construir o jogo **Peg Thing** aplicando tudo que aprendeu

## 🔄 Funções Puras: O Que São e Por Que

### Definição
Uma função é **pura** se atende dois critérios:

1. **Transparência Referencial**: Sempre retorna o mesmo resultado para os mesmos argumentos
2. **Sem Efeitos Colaterais**: Não pode fazer mudanças observáveis fora da função

### Transparência Referencial
```clojure
;; Funções puras (transparência referencial)
(+ 1 2)                    ; => 3 (sempre)
(defn wisdom [words]
  (str words ", Daniel-san"))

;; Funções não-puras (não são transparência referencial)
(defn year-end-evaluation []
  (if (> (rand) 0.5)
    "You get a raise!"
    "Better luck next year!"))

;; analyze-file não é pura, mas analysis é
(defn analyze-file [filename]
  (analysis (slurp filename)))

(defn analysis [text]
  (str "Character count: " (count text)))
```

### Sem Efeitos Colaterais
**JavaScript com efeitos colaterais:**
```javascript
var haplessObject = { emotion: "Carefree!" };

var evilMutator = function(object){
  object.emotion = "So emo :'(";  // EFEITO COLATERAL!
}

evilMutator(haplessObject);
haplessObject.emotion;  // => "So emo :'("
```

**Clojure sem efeitos colaterais:**
```clojure
;; Estruturas imutáveis - sem efeitos colaterais possíveis!
(def my-data {:emotion "Carefree!"})
(assoc my-data :emotion "So emo :'(")  ; => {:emotion "So emo :'("}
my-data                                  ; => {:emotion "Carefree!"} (inalterado!)
```

### Vantagens das Funções Puras
- ✅ **Fáceis de raciocinar**: Isoladas completamente
- ✅ **Consistentes**: Sem surpresas
- ✅ **Confiáveis**: Como aritmética
- ✅ **Composáveis**: Podem ser combinadas
- ✅ **Testáveis**: Entrada/saída previsível

## 🏗️ Vivendo com Estruturas de Dados Imutáveis

### Recursão ao Invés de for/while

**JavaScript mutável:**
```javascript
var wrestlers = getAlligatorWrestlers();
var totalBites = 0;
for(var i=0; i < wrestlers.length; i++){
  totalBites += wrestlers[i].timesBitten;  // MUTAÇÃO!
}
```

**Clojure imutável com recursão:**
```clojure
(defn sum
  ([vals] (sum vals 0))
  ([vals accumulating-total]
     (if (empty? vals)
       accumulating-total
       (recur (rest vals) (+ (first vals) accumulating-total)))))

;; Chamadas recursivas (conceptual)
(sum [39 5 1])
(sum [39 5 1] 0)
(sum [5 1] 39)
(sum [1] 44)  
(sum [] 45)   ; caso base
; => 45
```

**Versão otimizada com `recur`:**
```clojure
(defn sum
  ([vals] (sum vals 0))
  ([vals accumulating-total]
     (if (empty? vals)
       accumulating-total
       (recur (rest vals) (+ (first vals) accumulating-total)))))
```

### Composição de Funções ao Invés de Mutação

**Ruby com mutação:**
```ruby
class GlamourShotCaption
  def clean!
    text.trim!              # MUTAÇÃO!
    text.gsub!(/lol/, "LOL") # MUTAÇÃO!
  end
end
```

**Clojure com composição:**
```clojure
(require '[clojure.string :as s])

(defn clean [text]
  (s/replace (s/trim text) #"lol" "LOL"))

(clean "My boa constrictor is so sassy lol!  ")
; => "My boa constrictor is so sassy LOL!"
```

**Processo:**
1. `text` → `s/trim` → `"My boa constrictor is so sassy lol!"`
2. Resultado → `s/replace` → `"My boa constrictor is so sassy LOL!"`

## 😎 Coisas Legais com Funções Puras

### comp - Composição de Funções
```clojure
;; Compor inc e *
((comp inc *) 2 3)         ; => 7
;; Equivale a: (inc (* 2 3)) => (inc 6) => 7

;; Acessando atributos de personagem
(def character
  {:name "Smooches McCutes"
   :attributes {:intelligence 10
                :strength 4
                :dexterity 5}})

(def c-int (comp :intelligence :attributes))
(def c-str (comp :strength :attributes))
(def c-dex (comp :dexterity :attributes))

(c-int character)          ; => 10
(c-str character)          ; => 4

;; Slots de magia baseados em inteligência
(def spell-slots-comp (comp int inc #(/ % 2) c-int))
(spell-slots-comp character)  ; => 6

;; Implementação simples de comp para duas funções
(defn two-comp [f g]
  (fn [& args]
    (f (apply g args))))
```

### memoize - Cache de Resultados
```clojure
;; Função lenta
(defn sleepy-identity [x]
  (Thread/sleep 1000)
  x)

(sleepy-identity "Mr. Fantastico")  ; => demora 1 segundo
(sleepy-identity "Mr. Fantastico")  ; => demora 1 segundo (novamente)

;; Versão memoizada
(def memo-sleepy-identity (memoize sleepy-identity))

(memo-sleepy-identity "Mr. Fantastico")  ; => demora 1 segundo
(memo-sleepy-identity "Mr. Fantastico")  ; => instantâneo!
```

## 🎯 Projeto: Peg Thing

### Como Jogar
O Peg Thing é baseado no jogo de pinos triangular:

1. **Setup**: Tabuleiro triangular com buracos preenchidos com pinos
2. **Um buraco vazio** para começar
3. **Objetivo**: Remover o máximo de pinos possível
4. **Mecânica**: Saltar sobre pinos para removê-los

```
Inicial:           Após movimento:
   a0                 a0
  b0 c0              b0 c0  
 d0 e0 f0           d0 e- f0
g0 h0 i0 j0        g0 h- i0 j0
```

### Organização do Código
Quatro tarefas principais:

1. **Criar tabuleiro novo**
2. **Retornar tabuleiro com resultado do movimento**  
3. **Representar tabuleiro textualmente**
4. **Lidar com interação do usuário**

### Criando o Tabuleiro

**Estrutura de dados:**
```clojure
{1  {:pegged true, :connections {6 3, 4 2}},
 2  {:pegged true, :connections {9 5, 7 4}},
 ;; ... mais posições
 :rows 5}
```

- **Chaves numéricas**: posições no tabuleiro
- **`:pegged`**: tem pino ou não
- **`:connections`**: destino → posição saltada

**Números triangulares:**
```clojure
(defn tri*
  ([] (tri* 0 1))
  ([sum n]
     (let [new-sum (+ sum n)]
       (cons new-sum (lazy-seq (tri* new-sum (inc n)))))))

(def tri (tri*))
(take 5 tri)               ; => (1 3 6 10 15)

(defn triangular? [n]
  (= n (last (take-while #(>= n %) tri))))

(defn row-tri [n]
  (last (take n tri)))

(defn row-num [pos]
  (inc (count (take-while #(> pos %) tri))))
```

**Conectando posições:**
```clojure
(defn connect [board max-pos pos neighbor destination]
  (if (<= destination max-pos)
    (reduce (fn [new-board [p1 p2]]
              (assoc-in new-board [p1 :connections p2] neighbor))
            board
            [[pos destination] [destination pos]])
    board))

(defn connect-right [board max-pos pos]
  (let [neighbor (inc pos)
        destination (inc neighbor)]
    (if-not (or (triangular? neighbor) (triangular? pos))
      (connect board max-pos pos neighbor destination)
      board)))

;; connect-down-left e connect-down-right similares...
```

**Adicionando posições:**
```clojure
(defn add-pos [board max-pos pos]
  (let [pegged-board (assoc-in board [pos :pegged] true)]
    (reduce (fn [new-board connection-creation-fn]
              (connection-creation-fn new-board max-pos pos))
            pegged-board
            [connect-right connect-down-left connect-down-right])))

(defn new-board [rows]
  (let [initial-board {:rows rows}
        max-pos (row-tri rows)]
    (reduce (fn [board pos] (add-pos board max-pos pos))
            initial-board
            (range 1 (inc max-pos)))))
```

### Movendo Pinos

```clojure
(defn pegged? [board pos]
  (get-in board [pos :pegged]))

(defn remove-peg [board pos]
  (assoc-in board [pos :pegged] false))

(defn place-peg [board pos]
  (assoc-in board [pos :pegged] true))

(defn move-peg [board p1 p2]
  (place-peg (remove-peg board p1) p2))

(defn valid-moves [board pos]
  (into {}
        (filter (fn [[destination jumped]]
                  (and (not (pegged? board destination))
                       (pegged? board jumped)))
                (get-in board [pos :connections]))))

(defn valid-move? [board p1 p2]
  (get (valid-moves board p1) p2))

(defn make-move [board p1 p2]
  (if-let [jumped (valid-move? board p1 p2)]
    (move-peg (remove-peg board jumped) p1 p2)))

(defn can-move? [board]
  (some (comp not-empty (partial valid-moves board))
        (map first (filter #(get (second %) :pegged) board))))
```

### Renderização e Impressão

```clojure
(def alpha-start 97)
(def letters (map (comp str char) (range alpha-start 123)))

(defn render-pos [board pos]
  (str (nth letters (dec pos))
       (if (get-in board [pos :pegged])
         (colorize "0" :blue)
         (colorize "-" :red))))

(defn row-positions [row-num]
  (range (inc (or (row-tri (dec row-num)) 0))
         (inc (row-tri row-num))))

(defn row-padding [row-num rows]
  (let [pad-length (/ (* (- rows row-num) pos-chars) 2)]
    (apply str (take pad-length (repeat " ")))))

(defn render-row [board row-num]
  (str (row-padding row-num (:rows board))
       (clojure.string/join " " 
         (map (partial render-pos board)
              (row-positions row-num)))))

(defn print-board [board]
  (doseq [row-num (range 1 (inc (:rows board)))]
    (println (render-row board row-num))))
```

### Interação com Jogador

```clojure
(defn letter->pos [letter]
  (inc (- (int (first letter)) alpha-start)))

(defn get-input 
  ([] (get-input nil))
  ([default]
     (let [input (clojure.string/trim (read-line))]
       (if (empty? input)
         default
         (clojure.string/lower-case input)))))

(defn prompt-move [board]
  (println "\\nHere's your board:")
  (print-board board)
  (println "Move from where to where? Enter two letters:")
  (let [input (map letter->pos (characters-as-strings (get-input)))]
    (if-let [new-board (make-move board (first input) (second input))]
      (user-entered-valid-move new-board)
      (user-entered-invalid-move board))))

(defn user-entered-valid-move [board]
  (if (can-move? board)
    (prompt-move board)
    (game-over board)))

(defn user-entered-invalid-move [board]
  (println "\\n!!! That was an invalid move :(\\n")
  (prompt-move board))
```

### Início do Jogo

```clojure
(defn game-over [board]
  (let [remaining-pegs (count (filter :pegged (vals board)))]
    (println "Game over! You had" remaining-pegs "pegs left:")
    (print-board board)
    (println "Play again? y/n [y]")
    (let [input (get-input "y")]
      (if (= "y" input)
        (prompt-rows)
        (do
          (println "Bye!")
          (System/exit 0))))))

(defn prompt-empty-peg [board]
  (println "Here's your board:")
  (print-board board)
  (println "Remove which peg? [e]")
  (prompt-move (remove-peg board (letter->pos (get-input "e")))))

(defn prompt-rows []
  (println "How many rows? [5]")
  (let [rows (Integer. (get-input 5))
        board (new-board rows)]
    (prompt-empty-peg board)))

;; Para jogar:
(prompt-rows)
```

## 📚 Conceitos-Chave Resumidos

### 🎯 Funções Puras
- **Transparência referencial**: mesmo input = mesmo output
- **Sem efeitos colaterais**: não alteram mundo externo
- **Benefícios**: previsibilidade, testabilidade, composição
- **Memoização possível**: cache de resultados

### 🏗️ Estruturas Imutáveis
- **Recursão** ao invés de loops mutáveis
- **Composição de funções** ao invés de mutação de objetos
- **Derivação de dados** ao invés de modificação
- **Segurança**: dados originais preservados

### 😎 Técnicas Avançadas
- **comp**: composição elegante de funções
- **memoize**: cache automático para funções puras
- **Closures**: funções que capturam ambiente
- **Funções retornando funções**: factory pattern funcional

### 🎯 Padrões de Design Funcional
- **Separação de dados e funções**: mais flexibilidade
- **Programação para abstrações**: reutilização
- **Imutabilidade por padrão**: menos bugs
- **Composição sobre herança**: mais modular

## 💡 Principais Lições do Peg Thing

### ✅ **Modelagem sem Mutação**
- Estado do jogo representado como dados imutáveis
- Movimentos geram novos estados
- Histórico preservado automaticamente
- Debugger-friendly

### ✅ **Arquitetura em Camadas**  
- **Camada inferior**: lógica pura (tabuleiro, movimento, validação)
- **Camada superior**: efeitos colaterais (I/O, impressão)
- Separação clara de responsabilidades
- Testabilidade melhor

### ✅ **Decomposição em Funções Pequenas**
- Uma função = uma responsabilidade
- Nomes expressivos revelam intenção
- Facilita compreensão e manutenção
- Reutilização de componentes

### ✅ **Recursão como Padrão**
- Loops substituídos por recursão
- `recur` para otimização
- Funções mutuamente recursivas (prompt-move ↔ user-entered-valid-move)
- Elegância e clareza

## 🚀 Vantagens da Programação Funcional

### 🔒 **Confiabilidade**
- Sem mutação acidental
- Efeitos colaterais controlados
- Dados sempre consistentes
- Menos debugging

### 🧩 **Composabilidade**
- Funções pequenas se combinam
- Reutilização máxima
- Abstrações poderosas
- Expressividade alta

### 🧪 **Testabilidade**
- Entrada/saída previsível
- Sem setup complexo
- Mock desnecessário
- Testes determinísticos  

### 🚀 **Performance**
- Memoização automática
- Structural sharing
- Lazy evaluation
- Paralelização natural

## 💻 Exercícios Recomendados

1. **Implementar `attr`**: `(attr :intelligence)` como `(comp :intelligence :attributes)`
2. **Implementar `comp`**: versão própria da composição
3. **Implementar `assoc-in`**: dica - use `assoc` com `[k & ks]`
4. **Estudar `update-in`**: e depois implementar versão própria
5. **Implementar `map` com `reduce`**: exercício mental valioso

---

> **Próximo**: Capítulo 6 - Organizing Your Project

> **Dica**: Pratique pensamento funcional - é uma mudança de paradigma que vale a pena!