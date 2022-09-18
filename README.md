# IOS-interview
It is useful guide for future IOS interview 

Hi, in this file i collected many useful information, then can be halped for your technical interview

## Overview:
### 1. Value Semantics
### 2. Memmory management
### 3. Collections
### 4. Multithreading
### 5. Dispatching


## Swift Value Semantics
  В языке Swift имеется два вида значений:
  1) Ссылочный тип (классы, clousers)
  2) Тип значения (структуры, перечисления, типы)
  
Ссылочный тип - тип, который хранит в себе указатель на обьект (адресс этого обьекта на каком-то участке памяти), хранится на куче (динамическая память), при этом сам указатель хранится в стеке 
 
Тип значения - хранится на стеке (статическая память). Есть два сценария когда value type не будет хранится на стеке:
a) Когда внутри value type есть reference
b) когда type конформит какой-нибудь проток
Если размер value type слишком большой, то происходит inline optimization и он переходи на кучу, а в стеке остается только указатель на него
