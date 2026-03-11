---
title: "LRU Cache: Implementando um Algoritmo de Cache com Deque e HashMap"
date: 2026-01-28 09:00:00 -0300
categories: [Algoritmos, Estrutura de Dados]
tags: [algoritmos, estrutura de dados]
render_with_liquid: false
---

O cache é fundamental para performance, mas a memória é finita. Como decidir qual item remover quando o cache está lotado? O algoritmo mais popular é o **LRU (Least Recently Used)**: removemos o item que não é acessado há mais tempo. Vamos implementar um do zero!

## Velocidade de O(1)

Um bom cache precisa ser rápido. Tanto a inserção quanto a busca devem ser, idealmente, **O(1)**. Para isso, precisamos de uma combinação de duas estruturas de dados:
1.  **HashMap:** Para busca rápida pelo ID.
2.  **Doubly Linked List (ou Deque):** Para manter a ordem de uso e permitir remoção rápida das pontas.

## A Lógica

- Quando um item é **acessado** ou **inserido**, ele vai para a **frente** da fila (o mais recente).
- Quando o cache atinge o limite, removemos o item da **cauda** da fila (o menos recente).

## Implementação em Java

Embora o Java tenha o `LinkedHashMap` que já faz isso, entender a implementação com `Deque` é um exercício clássico de entrevista técnica.

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, V> map = new HashMap<>();
    private final Deque<K> queue = new LinkedList<>();

    public LRUCache(int capacity) {
        this.capacity = capacity;
    }

    public synchronized V get(K key) {
        if (!map.containsKey(key)) return null;
        
        // Move para a frente da fila (recente)
        queue.remove(key);
        queue.addFirst(key);
        return map.get(key);
    }

    public synchronized void put(K key, V value) {
        if (map.containsKey(key)) {
            queue.remove(key);
        } else if (map.size() >= capacity) {
            // Remove o mais antigo (cauda da fila)
            K oldest = queue.removeLast();
            map.remove(oldest);
        }
        
        map.put(key, value);
        queue.addFirst(key);
    }
}
```

## Por que Deque?

O `Deque` (Double Ended Queue) permite adicionar e remover itens de ambas as pontas em tempo constante. Em uma implementação real de alta performance, usaríamos uma `DoublyLinkedList` customizada para evitar o custo de busca do método `queue.remove(key)`, que é O(N). Com uma lista ligada própria, podemos guardar o nó no HashMap e removê-lo em O(1).

## Curiosidade: Onde isso é usado?

- **Redis:** Usa variantes de LRU para despejar chaves quando a memória acaba.
- **Sistemas Operacionais:** No gerenciamento de páginas de memória virtual.
- **Bancos de Dados:** No cache de páginas de disco (Buffer Pool).

## Conclusão

Implementar um LRU Cache é um excelente exercício para entender como combinar estruturas de dados para resolver problemas de performance. Se você quer algo pronto e performático em Java, use o `LinkedHashMap` com o construtor de `accessOrder` ou bibliotecas como o **Caffeine Cache**.