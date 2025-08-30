# Capítulo 9: The Sacred Art of Concurrent and Parallel Programming

## 🎯 Conceitos Fundamentais

### Concorrência vs Paralelismo
**Concorrência**: Gerenciar múltiplas tarefas ao mesmo tempo
**Paralelismo**: Executar múltiplas tarefas simultaneamente

### Analogia da Lady Gaga
```
Sequencial:    "Não posso textar com bebida na mão"
Concorrente:   "Vou soltar a bebida, textar, depois voltar a beber"
Paralelo:      "Posso textar com uma mão e beber com a outra"
```

### Blocking vs Assíncrono
- **Blocking**: Espera operação terminar (como aguardar resposta)
- **Assíncrono**: Continua outras tarefas enquanto aguarda

## 🧵 JVM Threads: Como Clojure Implementa

### O Que São Threads?
**Thread** = subprograma que executa instruções com acesso compartilhado ao estado

```
Single Thread:    [A1] → [A2] → [A3] → [A4]
Two Threads:      [A1] → [A2] → [B1] → [A3] → [B2] → [A4]
                         ↑ intercalação ↑
```

### Execução em Multicore
```
Core 1: Thread A  [A1] → [A2] → [A3] → [A4]
Core 2: Thread B  [B1] → [B2] → [B3] → [B4]
                  ↑ execução paralela ↑
```

## 👹 Os Três Goblins da Concorrência

### 1. Reference Cell Problem
**Problema**: Duas threads lendo/escrevendo no mesmo local

```
Thread A: WRITE X = 0 → READ X → WRITE X = X + 1
Thread B:                READ X → WRITE X = X + 1

Ordem 1: A1, A2, A3, B1, B2 → X = 2 ✓
Ordem 2: A1, A2, B1, A3, B2 → X = 1 ✗ (perdeu uma operação)
```

### 2. Mutual Exclusion
**Problema**: Acesso simultâneo a recursos compartilhados

```
Thread A escreve: "Pelo poder investido em mim"
Thread B escreve: "Trovão, raio, vento e chuva"

Resultado intercalado:
"Pelo poder investido Trovão, raio em mim, vento pela chuva"
```

### 3. Dwarven Berserkers (Deadlock)
**Cenário**: 4 anões, 4 machados, ritual requer 2 machados

```
Anão 1: pega machado esquerdo → espera direito
Anão 2: pega machado esquerdo → espera direito  
Anão 3: pega machado esquerdo → espera direito
Anão 4: pega machado esquerdo → espera direito
→ DEADLOCK! Todos esperando infinitamente
```

## 🚀 Futures, Delays e Promises

### Problema do Código Serial
Código tradicional acopla 3 eventos:
1. **Definição** da tarefa
2. **Execução** da tarefa  
3. **Requisição** do resultado

```clojure
;; Tudo acontece junto, sequencialmente
(web-api/get :dwarven-beard-waxes)  ; define + executa + espera resultado
```

### Futures: Execução Assíncrona

```clojure
;; Define + executa em thread separada, resultado depois
(future (Thread/sleep 4000)
        (println "Imprime após 4 segundos"))
(println "Imprime imediatamente")

;; Obtendo resultado
(let [result (future (println "executa uma vez")
                     (+ 1 1))]
  (println "deref:" (deref result))  ; primeira vez
  (println "@:" @result))            ; segunda vez (cached)
; => "executa uma vez" (só uma vez!)
; => deref: 2
; => @: 2

;; Com timeout
(deref (future (Thread/sleep 1000) 42) 10 :timeout)
; => :timeout (não esperou 1 segundo)

;; Verificando se terminou
(realized? (future (Thread/sleep 1000)))  ; => false
```

### Delays: Definição Sem Execução

```clojure
;; Define tarefa SEM executar
(def jackson-5-delay
  (delay (let [message "Just call my name"]
           (println "Executando delay!")
           message)))

;; Nada foi executado ainda...

;; Força execução
(force jackson-5-delay)
; => "Executando delay!"
; => "Just call my name"

;; Segunda chamada usa cache
@jackson-5-delay  ; => "Just call my name" (sem println)
```

### Exemplo Prático: Upload de Fotos
```clojure
(def gimli-headshots ["serious.jpg" "fun.jpg" "playful.jpg"])

(defn email-user [email] 
  (println "Enviando notificação para" email))

(defn upload-document [headshot] 
  (println "Upload" headshot) 
  true)

;; Notifica apenas no primeiro upload concluído
(let [notify (delay (email-user "gimli@axe.com"))]
  (doseq [headshot gimli-headshots]
    (future 
      (upload-document headshot)
      (force notify))))  ; Só executa uma vez!
```

### Promises: Resultado Sem Tarefa

```clojure
;; Cria promise vazia
(def my-promise (promise))

;; Entrega valor em outro momento
(deliver my-promise (+ 1 2))

;; Obtém resultado
@my-promise  ; => 3

;; Só pode entregar uma vez
(deliver my-promise 999)  ; Ignorado!
@my-promise  ; => 3 (ainda)
```

### Exemplo: Primeira Manteiga de Yak Satisfatória
```clojure
(def yak-butter-products
  [{:store "Yak Butter Intl" :price 90 :smoothness 90}
   {:store "Butter Than Nothing" :price 150 :smoothness 83}  
   {:store "Baby Got Yak" :price 94 :smoothness 99}])

(defn satisfactory? [butter]
  (and (<= (:price butter) 100)
       (>= (:smoothness butter) 97)
       butter))

;; Versão sequencial - 3+ segundos
(time (some (comp satisfactory? mock-api-call) yak-butter-products))

;; Versão paralela com promise - ~1 segundo  
(time
  (let [butter-promise (promise)]
    (doseq [butter yak-butter-products]
      (future 
        (when-let [good-butter (satisfactory? (mock-api-call butter))]
          (deliver butter-promise good-butter))))
    @butter-promise))
```

### Promises como Callbacks
```clojure
(let [wisdom-promise (promise)]
  ;; Future bloqueia esperando promise
  (future (println "Sabedoria Ferengi:" @wisdom-promise))
  
  ;; Após delay, entrega valor
  (Thread/sleep 100)
  (deliver wisdom-promise "Sussurre seu caminho ao sucesso"))
; => "Sabedoria Ferengi: Sussurre seu caminho ao sucesso"
```

## 🎯 Exemplo Avançado: Queue Customizada

### Problema: Serialização de Partes Críticas
Estratégia: Tarefas concorrentes + parte serial ordenada

```clojure
;; Macro helper para sleep
(defmacro wait [timeout & body]
  `(do (Thread/sleep ~timeout) ~@body))

;; Saudação britânica em ordem correta
(-> (enqueue saying (wait 200 "'Ello, gov'na!") (println @saying))
    (enqueue saying (wait 400 "Pip pip!") (println @saying))  
    (enqueue saying (wait 100 "Cheerio!") (println @saying)))
```

### Implementação do enqueue
```clojure
(defmacro enqueue
  ;; Versão completa com queue
  ([q concurrent-promise-name concurrent serialized]
   `(let [~concurrent-promise-name (promise)]
      (future (deliver ~concurrent-promise-name ~concurrent))
      (deref ~q)  ; Espera queue anterior
      ~serialized
      ~concurrent-promise-name))
  
  ;; Versão inicial sem queue
  ([concurrent-promise-name concurrent serialized]
   `(enqueue (future) ~concurrent-promise-name ~concurrent ~serialized)))
```

### Como Funciona
1. **Cria promise** para cada tarefa
2. **Future executa** parte concorrente 
3. **Deref espera** queue anterior terminar
4. **Executa parte serial** em ordem
5. **Retorna promise** para próxima etapa

## 📚 Conceitos-Chave Resumidos

### 🧵 Threading Fundamentals
- **Thread**: Subprograma com estado compartilhado
- **Interleaving**: Alternância entre threads (single-core)
- **Parallelism**: Execução simultânea (multi-core)
- **Nondeterministic**: Ordem de execução imprevisível

### 👹 Concurrency Goblins
- **Reference Cell**: Race conditions em variáveis compartilhadas
- **Mutual Exclusion**: Acesso conflitante a recursos
- **Deadlock**: Espera circular infinita por recursos

### 🚀 Clojure Tools
- **Future**: Define + executa agora, resultado depois
- **Delay**: Define agora, executa + resultado depois  
- **Promise**: Resultado esperado, sem definir como obter
- **Deref (@)**: Força obtenção de resultado (pode bloquear)

### 🎯 Padrões de Uso
- **Futures**: Tarefas paralelas independentes
- **Delays**: Inicialização lazy, operações únicas
- **Promises**: Coordenação entre threads, callbacks
- **Combining**: Patterns complexos de sincronização

## 💡 Melhores Práticas

### ✅ Do
- Use **futures** para I/O paralelo (chamadas API, arquivos)
- Use **delays** para computações caras executadas uma vez
- Use **promises** para coordenação entre threads
- **Combine ferramentas** para padrões complexos
- **Timeout** em operações que podem bloquear
- **Cache resultados** automaticamente (future/delay)

### ❌ Don't
- Evite **estado mutável compartilhado** sem proteção
- Não **ignore timeouts** em operações blocking
- Não **misture threading** com efeitos colaterais sem cuidado
- Evite **deadlock** por design (ordem de acquisição)

### 🔧 Debugging Dicas
- **realized?** verifica se future/delay terminou
- **time** mede performance de operações paralelas
- **Thread/sleep** para simular operações lentas
- **deref com timeout** evita bloqueios infinitos

## 🚀 Exercícios Recomendados

1. **Search Engine**: Função que busca termo no Bing + Google usando `slurp`
2. **Configurable Engines**: Versão que aceita lista de engines como parâmetro  
3. **URL Extractor**: Extrai URLs dos resultados usando regex/parsing

---

> **Próximo**: Capítulo 10 - Atoms, Refs, Vars, and Cuddle Zombies

> **Lição Principal**: Concorrência = separar definição, execução e resultado. Tools simples → patterns poderosos!