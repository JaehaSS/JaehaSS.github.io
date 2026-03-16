---
layout: post
title: "Iterator 와 Generator"
date: 2025-01-19
description: "Python의 Iterator와 Generator의 개념, 사용법, 메모리 비교를 통해 차이점과 공통점을 정리한 글입니다."
tags: [Python]
tistory_id: 50
---

# Iterator 와 Generator

## Iterator

- 반복 가능한 객체로, `__iter__()` 와 `__next__()` 메서드를 구현하는 객체
- 리스트, 튜플, 집합, 딕셔너리, `__iter__`, `__next__` 메서드가 존재하는 객체

### 예제

```python
# list
list_data = [1,2,3,4,5]
for i in list_data:
    print(i, end=" ") # 1 2 3 4 5

# tuple
tuples_data = (1,2,3,4,5)
for i in tuple_data:
    print(i, end=" ") # 1 2 3 4 5

# set
set_data = {1,2,3,4,5}
for i in set_data:
    print(i, end=" ") # 1 2 3 4 5

# dict
dict_data = {1 : "one", 2 : "two" }
for i in dict_data:
    print(i, end=" ")  # 1 2

# __iter__, __next__ 구현한 Class
class IterTest:
    def __init__(self):
        self.start = 1
        self.end = 5
        self.current = self.start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        else:
            v = self.current
            self.current += 1
            return v

iter_test = IterTest()
for i in iter_test:
    print(i, end=" ") # 1 2 3 4 5
```

## Generator

- 이터레이터의 일종임, 값을 필요로 할 때마다 동적으로 생성하는 함수
- 함수 안에 yield 키워드를 사용함
- yield를 사용하여 값을 반환하고 함수의 실행 상태를 일시 중단
- 객체를 호출할 때마다 값을 순차적으로 호출함

```python
def number_print():
    yield 1
    yield 2
    yield 3
    yield 4
    yield 5

call_func = number_print()
print(next(call_func)) # 1
print(next(call_func)) # 2
print(next(call_func)) # 3
print(next(call_func)) # 4
print(next(call_func)) # 5
print(next(call_func)) # raise StopIteration
```

## 메모리 비교

```python
from sys import getsizeof # Memory 크기 확인

generator_data = (x for x in range(1,6)) # Generator 생성
iterator_data = [x for x in range(1,6)] # iterator 생성

print(getsizeof(generator_data)) # 112
print(getsizeof(iterator_data)) # 120
```

iterator 보다 generator 가 크기가 더 작은 걸 확인할 수 있다.

## 정리

공통점으로는 iterator 와 generator 는 반복할 수 있는 객체이고 값을 순차적으로 반환한다.

차이점으로는 generator는 함수 내에 yield 키워드를 사용하고 메모리를 효율적으로 사용한다.
