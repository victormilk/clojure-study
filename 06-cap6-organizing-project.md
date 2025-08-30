# Capítulo 6: Organizing Your Project

## 🎯 Conceito Central: Projeto como Biblioteca

**Namespaces** são como catálogos de biblioteca - associam **símbolos** (nomes legíveis) a **vars** (referências para endereços onde objetos estão armazenados).

```clojure
(ns-name *ns*)          ; => user
(ns-interns *ns*)       ; => mapa de símbolos para vars
```

## 🗂️ Armazenando Objetos com def

```clojure
(def great-books ["East of Eden" "The Glass Bead Game"])
; => #'user/great-books

;; Processo interno:
;; 1. Atualiza namespace atual com associação símbolo→var
;; 2. Encontra prateleira livre
;; 3. Armazena vetor na prateleira
;; 4. Escreve endereço da prateleira no var
;; 5. Retorna o var

(deref #'user/great-books)  ; Acessar via var
; => ["East of Eden" "The Glass Bead Game"]

great-books                 ; Acesso normal (símbolo resolve para var, deref automático)
; => ["East of Eden" "The Glass Bead Game"]
```

**⚠️ Problema**: Name collision ao redefinir mesmo símbolo!

## 🏗️ Criando e Mudando Namespaces

### create-ns e in-ns
```clojure
(create-ns 'cheese.taxonomy)           ; Cria namespace
(in-ns 'cheese.analysis)              ; Cria + muda para namespace

;; Símbolo totalmente qualificado
cheese.taxonomy/cheddars              ; namespace/nome
```

### refer - Controle Fino
```clojure
(in-ns 'cheese.analysis)
(clojure.core/refer 'cheese.taxonomy)  ; Importa símbolos

;; Com filtros
(refer 'cheese.taxonomy :only ['bries])           ; Apenas bries
(refer 'cheese.taxonomy :exclude ['bries])        ; Tudo exceto bries  
(refer 'cheese.taxonomy :rename {'bries 'yummy-bries})  ; Renomear

;; Funções privadas
(defn- private-function [])            ; Não acessível de outros namespaces
```

### alias - Símbolos Qualificados Mais Curtos
```clojure
(alias 'taxonomy 'cheese.taxonomy)
taxonomy/bries                         ; Usar alias curto
; => ["Wisconsin" "Somerset" "Brie de Meaux" "Brie de Melun"]
```

## 📁 Organização Real de Projetos

### Relacionamento Arquivo ↔ Namespace
```
Regras de Mapeamento:
- Raiz do código: src/ (padrão)
- Hífens → underscores (the-divine-cheese → the_divine_cheese)
- Pontos → diretórios (cheese.analysis → cheese/analysis/)
- Componente final → arquivo .clj (core → core.clj)

Exemplo:
the-divine-cheese-code.visualization.svg
↓
src/the_divine_cheese_code/visualization/svg.clj
```

### require e use
```clojure
;; require: carrega namespace
(require 'the-divine-cheese-code.visualization.svg)

;; Com alias
(require '[the-divine-cheese-code.visualization.svg :as svg])

;; use: require + refer (não recomendado em produção)
(use 'the-divine-cheese-code.visualization.svg)
```

### Macro ns - Forma Padrão
```clojure
(ns the-divine-cheese-code.core
  ;; Controlar o que vem de clojure.core
  (:refer-clojure :exclude [println])
  
  ;; Require com alias e refer
  (:require [clojure.java.browse :as browse]
            [the-divine-cheese-code.visualization.svg :refer [xml]])
  
  ;; Use (evitar)
  (:use [clojure.java.browse :only [browse-url]])
  
  (:gen-class))

;; Equivale a:
(in-ns 'the-divine-cheese-code.core)
(require '[clojure.java.browse :as browse])
(refer 'the-divine-cheese-code.visualization.svg :only ['xml])
```

## 🕵️ Projeto: Caçar o Ladrão de Queijo

**SVG de mapa mostrando localização de roubos:**

```clojure
;; Dados dos roubos
(def heists [{:location "Cologne, Germany"
              :cheese-name "Archbishop Hildebold's Cheese Pretzel"
              :lat 50.95 :lng 6.97}
             ;; ... mais roubos
             ])

;; Funções de transformação
(defn translate-to-00 [locations]
  (let [mincoords (min locations)]
    (map #(merge-with - % mincoords) locations)))

(defn scale [width height locations]
  (let [maxcoords (max locations)
        ratio {:lat (/ height (:lat maxcoords))
               :lng (/ width (:lng maxcoords))}]
    (map #(merge-with * % ratio) locations)))

;; Pipeline de transformação
(defn transform [width height locations]
  (->> locations
       translate-to-00
       (scale width height)))

;; Geração SVG
(defn xml [width height locations]
  (str "<svg height=\\"" height "\\" width=\\"" width "\\">"
       (-> (transform width height locations)
           points
           line)
       "</svg>"))
```

**Resultado**: Padrão dos roubos forma um lambda (λ) - era Clojure o tempo todo! 🤯

## 📚 Conceitos-Chave

### 🏛️ Arquitetura de Namespaces
- **Símbolos**: Nomes que você usa no código
- **Vars**: Referências para localizações de memória  
- **Namespaces**: Mapas símbolo→var + contexto atual
- **Fully qualified**: namespace/símbolo

### 🔧 Ferramentas de Organização
- **def**: Armazena objetos, cria var
- **require**: Carrega namespace
- **refer**: Importa símbolos sem qualificação  
- **alias**: Abreviações para namespaces
- **use**: require + refer (evitar)
- **ns**: Macro master para setup

### 📂 Estrutura de Projeto
- **src/**: Código fonte (padrão Leiningen)
- **Mapeamento 1:1**: namespace ↔ arquivo
- **Convenções**: hífens→underscores, pontos→diretórios
- **Privacidade**: defn- para funções privadas

### ⚠️ Problemas e Soluções
- **Name collision**: Use namespaces separados
- **Import explícito**: require sempre necessário
- **Organize por domínio**: não por tipo de arquivo
- **Evite use**: prefira require com refer seletivo

## 💡 Melhores Práticas

### ✅ Do
- Use namespaces para evitar conflitos
- Organize por funcionalidade/domínio
- require explicitamente o que precisa
- Prefira require com :as ou :refer seletivo
- Use defn- para funções internas

### ❌ Don't  
- Não use (:use) em produção - muito amplo
- Não assuma que namespaces são carregados automaticamente
- Não misture lógicas diferentes no mesmo namespace
- Não ignore convenções de nomeação arquivo→namespace

---

> **Próximo**: Capítulo 7 - Reading, Evaluation, and Macros