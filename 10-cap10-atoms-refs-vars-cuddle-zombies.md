# Capítulo 10: Atoms, Refs, Vars, and Cuddle Zombies

## 🧠 Metafísica: OO vs Clojure

### Metafísica OO: Objetos Mutáveis
**Problema**: Objeto como célula de referência compartilhada

```ruby
# Ruby - estado mutável compartilhado
class CuddleZombie
  attr_accessor :cuddle_hunger_level, :percent_deteriorated
  
  def initialize(hunger = 1, decay = 0)
    self.cuddle_hunger_level = hunger
    self.percent_deteriorated = decay  
  end
end

fred = CuddleZombie.new(2, 3)
fred.cuddle_hunger_level = 5  # MUTAÇÃO!
```

**Problemas de Concorrência**:
```ruby
# Race condition
if fred.percent_deteriorated >= 50
  Thread.new { logger.log(fred.cuddle_hunger_level) }  # Pode logar valor inconsistente!
end

# Mutual exclusion  
fred.cuddle_hunger_level += 1
# Outra thread pode ler estado inconsistente aqui
fred.percent_deteriorated += 1
```

### Metafísica Clojure: Sucessão de Valores
**Conceito**: Identidade = nome para sucessão de valores imutáveis

```
Estado não é objeto que muda, mas valores em sequência:
F1 → [processo] → F2 → [processo] → F3 → [processo] → F4

Fred = nome para a sequência F1, F2, F3, F4...
Estado = valor de uma identidade num ponto no tempo
```

**Analogia**: Número de telefone do Alan
- "Número do Alan" = identidade
- Números específicos ao longo do tempo = estados
- Cada número = valor imutável

## ⚛️ Atoms: Estado Simples e Seguro

### Criação e Uso Básico
```clojure
;; Criando atom
(def fred (atom {:cuddle-hunger-level 0
                 :percent-deteriorated 0}))

;; Lendo estado atual (nunca bloqueia)
@fred
; => {:cuddle-hunger-level 0, :percent-deteriorated 0}

;; Estado é imutável - seguro para threads
(let [zombie-state @fred]
  (if (>= (:percent-deteriorated zombie-state) 50)
    (future (println (:cuddle-hunger-level zombie-state)))))
; Sem race conditions!
```

### Atualizações com swap!
```clojure
;; swap! usa compare-and-set semânticas
(swap! fred
       (fn [current-state]
         (merge-with + current-state {:cuddle-hunger-level 1})))
; => {:cuddle-hunger-level 1, :percent-deteriorated 0}

;; Função com múltiplos argumentos
(defn increase-cuddle-hunger [zombie-state increase-by]
  (merge-with + zombie-state {:cuddle-hunger-level increase-by}))

(swap! fred increase-cuddle-hunger 10)
; => {:cuddle-hunger-level 11, :percent-deteriorated 0}

;; Usando funções built-in
(swap! fred update-in [:cuddle-hunger-level] + 5)
; => {:cuddle-hunger-level 16, :percent-deteriorated 0}
```

### Compare-and-Set Interno
```
swap! internamente:
1. Lê estado atual do atom
2. Aplica função ao estado
3. Verifica se estado inicial ainda é atual  
4. Se sim: atualiza atom
5. Se não: retry (volta ao passo 1)

→ Nunca perde atualizações!
```

### reset! para Valores Absolutos
```clojure
;; Define estado absolutamente (sem verificar atual)
(reset! fred {:cuddle-hunger-level 0
              :percent-deteriorated 0})
```

### Mantendo Estados Passados
```clojure
(let [num (atom 1)
      s1 @num]        ; Captura Estado 1
  (swap! num inc)     ; Cria Estado 2
  (println "Estado 1:" s1)
  (println "Estado atual:" @num))
; => Estado 1: 1
; => Estado atual: 2
```

## 👁️ Watches e Validators

### Watches: Observando Mudanças
```clojure
(defn shuffle-speed [zombie]
  (* (:cuddle-hunger-level zombie)
     (- 100 (:percent-deteriorated zombie))))

;; Watch function (4 argumentos obrigatórios)
(defn shuffle-alert
  [key watched old-state new-state]
  (let [sph (shuffle-speed new-state)]
    (if (> sph 5000)
      (println "CORRA! SPH:" sph "Fonte:" key)
      (println "Tudo bem com" key "SPH:" sph))))

;; Adicionando watch
(reset! fred {:cuddle-hunger-level 22 :percent-deteriorated 2})
(add-watch fred :fred-alert shuffle-alert)

(swap! fred update-in [:cuddle-hunger-level] + 30)
; => CORRA! SPH: 5044 Fonte: :fred-alert
```

### Validators: Controlando Estados Válidos
```clojure
;; Validator function (1 argumento)
(defn percent-deteriorated-validator
  [{:keys [percent-deteriorated]}]
  (and (>= percent-deteriorated 0)
       (<= percent-deteriorated 100)))

;; Atom com validator
(def bobby
  (atom {:cuddle-hunger-level 0 :percent-deteriorated 0}
        :validator percent-deteriorated-validator))

(swap! bobby update-in [:percent-deteriorated] + 200)
; => Exception: Invalid reference state

;; Validator com exception customizada
(defn percent-validator-with-error
  [{:keys [percent-deteriorated]}]
  (or (and (>= percent-deteriorated 0)
           (<= percent-deteriorated 100))
      (throw (IllegalStateException. "Valor inválido!"))))
```

## 🧦 Refs: Coordenação de Estado Múltiplo

### Problema: Transferência de Meias
```clojure
;; Problema: Gnomo rouba meia do secador
;; Deve ser atômico: secador perde + gnomo ganha simultaneamente
;; Nunca: meia em ambos, meia em nenhum

(def sock-varieties
  #{"darned" "argyle" "wool" "horsehair" "mulleted"
    "passive-aggressive" "striped" "business"})

(defn sock-count [variety count]
  {:variety variety :count count})

(defn generate-sock-gnome [name]
  {:name name :socks #{}})

;; Criando refs (não atoms!)
(def sock-gnome (ref (generate-sock-gnome "Barumpharumph")))
(def dryer (ref {:name "LG 1337"
                 :socks (set (map #(sock-count % 2) sock-varieties))}))
```

### Transações STM com dosync
```clojure
(defn steal-sock [gnome dryer]
  (dosync  ; Inicia transação
   (when-let [pair (some #(if (= (:count %) 2) %) (:socks @dryer))]
     (let [updated-count (sock-count (:variety pair) 1)]
       (alter gnome update-in [:socks] conj updated-count)  ; Gnomo ganha
       (alter dryer update-in [:socks] disj pair)           ; Remove par
       (alter dryer update-in [:socks] conj updated-count)))))  ; Adiciona 1

(steal-sock sock-gnome dryer)

(:socks @sock-gnome)
; => #{{:variety "passive-aggressive", :count 1}}
```

### Características das Transações
**ACID Properties**:
- **Atomic**: Todos refs atualizados ou nenhum
- **Consistent**: Refs sempre em estado válido  
- **Isolated**: Transações como se fossem seriais

### Estado In-Transaction
```clojure
(def counter (ref 0))

(future
  (dosync
   (alter counter inc)
   (println @counter)        ; => 1 (dentro da transação)
   (Thread/sleep 500)
   (alter counter inc)
   (println @counter)))      ; => 2 (dentro da transação)

(Thread/sleep 250)
(println @counter)           ; => 0 (fora da transação)
```

### commute vs alter
**alter**: Força retry se ref mudou
```clojure
;; Se outro transaction mudou o ref, retry
(alter ref-name update-fn)
```

**commute**: Nunca força retry, reaplica função no commit
```clojure
;; Mais performático, mas função deve ser comutativa
(defn sleep-print-update [sleep-time thread-name update-fn]
  (fn [state]
    (Thread/sleep sleep-time)
    (println (str thread-name ": " state))
    (update-fn state)))

(def counter (ref 0))
(future (dosync (commute counter (sleep-print-update 100 "A" inc))))
(future (dosync (commute counter (sleep-print-update 150 "B" inc))))

; Thread A: 0 | 100ms  
; Thread B: 0 | 150ms
; Thread A: 0 | 200ms  <- função reaplicada com estado atual
; Thread B: 1 | 300ms  <- resultado final: 2
```

### Quando commute é Perigoso
```clojure
;; PERIGOSO: função deriva de estado que pode mudar
(def giver (ref #{1}))
(def receiver-a (ref #{}))  
(def receiver-b (ref #{}))

;; Duas transactions tentam pegar o mesmo item
(future (dosync (let [gift (first @giver)]
                  (commute receiver-a conj gift)
                  (commute giver disj gift))))

(future (dosync (let [gift (first @giver)]  
                  (commute receiver-b conj gift)
                  (commute giver disj gift))))

;; Resultado: gift duplicado! ❌
@receiver-a  ; => #{1}  
@receiver-b  ; => #{1}
@giver       ; => #{}
```

## 🔄 Vars: Binding Dinâmico e Configuração

### Dynamic Vars
```clojure
;; Criando var dinâmica (note os asteriscos)
(def ^:dynamic *notification-address* "dobby@elf.org")

;; Binding temporário
(binding [*notification-address* "test@elf.org"]
  *notification-address*)
; => "test@elf.org"

;; Função que usa var dinâmica
(defn notify [message]
  (str "TO: " *notification-address* "\n"
       "MESSAGE: " message))

;; Normal
(notify "Help!")
; => "TO: dobby@elf.org\nMESSAGE: Help!"

;; Com binding para testes
(binding [*notification-address* "test@elf.org"]
  (notify "Test message"))
; => "TO: test@elf.org\nMESSAGE: Test message"
```

### Vars Built-in do Clojure
```clojure
;; *out* para output
(binding [*out* (clojure.java.io/writer "output.txt")]
  (println "Vai para arquivo, não REPL"))

;; *print-length* para coleções
(println [1 2 3 4 5])         ; => [1 2 3 4 5]

(binding [*print-length* 2]
  (println [1 2 3 4 5]))       ; => [1 2 ...]
```

### set! em Dynamic Vars
```clojure
(def ^:dynamic *troll-thought* nil)

(defn troll-riddle [your-answer]
  (let [number "man meat"]
    (when (thread-bound? #'*troll-thought*)  ; Verifica se bound
      (set! *troll-thought* number))          ; Set valor
    (if (= number your-answer)
      "Pode passar!"
      "Hora de te comer!")))

(binding [*troll-thought* nil]
  (println (troll-riddle 2))
  (println "Resposta era:" *troll-thought*))
; => Hora de te comer!
; => Resposta era: man meat
```

### Per-Thread Binding
```clojure
;; Thread manual NÃO herda bindings
(.start (Thread. #(.write *out* "vai para stdout")))

;; bound-fn propaga bindings
(.start (Thread. (bound-fn [] (.write *out* "vai para REPL"))))

;; Futures automaticamente propagam bindings
(future (.write *out* "vai para REPL"))
```

### alter-var-root
```clojure
(def power-source "hair")
(alter-var-root #'power-source (fn [_] "7-eleven parking lot"))
power-source
; => "7-eleven parking lot"

;; with-redefs para mudanças temporárias (inclusive em threads filhas)
(with-redefs [*out* *out*]
  (doto (Thread. #(println "aparece no REPL"))
    .start
    .join))
```

## ⚡ pmap: Paralelização Sem Estado

### Comparando map vs pmap
```clojure
;; Gerando dados de teste
(def alphabet-length 26)
(def letters (mapv (comp str char (partial + 65)) (range alphabet-length)))

(defn random-string [length]
  (apply str (take length (repeatedly #(rand-nth letters)))))

(def orc-names (doall (take 3000 (repeatedly #(random-string 7000)))))

;; Comparação de performance
(time (dorun (map clojure.string/lower-case orc-names)))
; => "Elapsed time: 270.182 msecs"

(time (dorun (pmap clojure.string/lower-case orc-names)))  
; => "Elapsed time: 147.562 msecs"  ; ~1.8x mais rápido!
```

### Problema: Overhead de Paralelização
```clojure
;; Tarefas pequenas = overhead > benefício
(def small-names (doall (take 20000 (repeatedly #(random-string 300)))))

(time (dorun (map clojure.string/lower-case small-names)))
; => "Elapsed time: 78.23 msecs"

(time (dorun (pmap clojure.string/lower-case small-names)))
; => "Elapsed time: 124.727 msecs"  ; 1.6x MAIS LENTO!
```

### Solução: Grain Size (Tamanho do Grão)
```clojure
;; Aumentar grain size com partition-all
(time
 (dorun
  (apply concat
         (pmap (fn [name-group] 
                 (doall (map clojure.string/lower-case name-group)))
               (partition-all 1000 small-names)))))
; => "Elapsed time: 44.677 msecs"  ; Muito melhor!
```

### ppmap: Partitioned pmap Genérica
```clojure
(defn ppmap
  "pmap particionado para tornar overhead compensador"
  [grain-size f & colls]
  (apply concat
   (apply pmap
          (fn [& pgroups] (doall (apply map f pgroups)))
          (map (partial partition-all grain-size) colls))))

(time (dorun (ppmap 1000 clojure.string/lower-case small-names)))
; => "Elapsed time: 44.902 msecs"
```

## 📚 Conceitos-Chave Resumidos

### 🧠 Metafísica Clojure
- **Value**: Entidade atômica, imutável (como números)
- **State**: Valor de uma identidade em um ponto no tempo
- **Identity**: Nome para sucessão de valores
- **Process**: Produz novos valores a partir de valores existentes

### ⚛️ Reference Types
- **Atom**: Estado simples, independente (compare-and-set)
- **Ref**: Coordenação de múltiplos estados (STM transactions)
- **Var**: Binding dinâmico, configuração global

### 🔧 Operações Principais
- **swap!**: Atualiza atom com função
- **reset!**: Define atom com valor absoluto
- **dosync + alter**: Transação coordenada para refs  
- **dosync + commute**: Como alter mas nunca retry
- **binding**: Binding temporário de vars dinâmicas

### ⚡ Concorrência Stateless
- **pmap**: map paralelo
- **ppmap**: pmap com grain size customizado
- **Grain size**: Quantidade de trabalho por thread

## 💡 Melhores Práticas

### ✅ Do
- **Atoms**: Para estado independente, simples
- **Refs**: Para coordenação de múltiplos estados
- **Vars dinâmicas**: Para configuração e recursos
- **pmap**: Para transformações independentes e computação pesada
- **Increase grain size**: Quando pmap é mais lento que map

### ❌ Don't
- Use **commute** apenas se função for realmente comutativa
- Não **alter-var-root** para assignment simples
- **pmap** com tarefas muito pequenas (overhead > benefício)
- **set!** vars não dinâmicas ou fora de binding

### 🎯 Quando Usar O Quê
- **Contador simples**: Atom com swap!
- **Transferência conta bancária**: Refs com dosync + alter
- **Configuração de teste**: Dynamic var com binding  
- **Map em lista grande**: pmap ou ppmap
- **Observar mudanças**: Watches
- **Validar estados**: Validators

---

> **Próximo**: Capítulo 11 - core.async

> **Lição Principal**: Estado = valor + tempo. Reference types = identidade segura. Cada tipo para um uso específico!