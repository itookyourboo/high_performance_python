# Списки и кортежи

Список - динамический массив.<br>
Кортеж - статический массив.

Память компьютера может быть представлена в виде последовательности пронумерованных ячеек (_bucket_), каждая из которых содержит число. Python хранит данные в этих ячейках в виде ссылок (_reference_), поэтому ячейки могут хранить любой тип данных.

Когда мы хотим создать массив, мы выделяем блок памяти - обращаемся к ядру и делаем запрос на использование `N` последовательных ячеек (_bucket_).

Массивы также хранят свой размер. Замечательная особенность массивов в том, что, зная начальный адрес, мы можем получить любую ячейку за `O(1)`.

```ipython
In [3]: x = list(range(10_000))
In [4]: %timeit x[5_000]
23.2 ns ± 0.1 ns per loop (mean ± std. dev. of 7 runs, 10,000,000 loops each)

In [5]: x = list(range(10_000_000))
In [6]: %timeit x[5_000_000]
22.8 ns ± 0.0411 ns per loop (mean ± std. dev. of 7 runs, 10,000,000 loops each)
```

### Поиск элемента в массиве

Если элемент можно узнать по индексу за `O(1)`, то индекс по элементу находится за `O(N)` в общем случае и за `O(logN)`, если известно, что массив отсортирован.

#### Линейный поиск - O(N)

```python
def linear_search(needle, array):
    for i, item in enumerate(array):
        if item == needle:
            return i
    return -1
```

```
Value 1        found in haystack of size 10000    at index 1        in 1.90920e-07 seconds
Value 6000     found in haystack of size 10000    at index 6000     in 2.74547e-04 seconds
Value 9000     found in haystack of size 10000    at index 9000     in 4.17478e-04 seconds
Value 1000000  found in haystack of size 10000    at index -1       in 4.46857e-04 seconds
Value 1        found in haystack of size 100000   at index 1        in 1.95215e-07 seconds
Value 6000     found in haystack of size 100000   at index 6000     in 2.67842e-04 seconds
Value 9000     found in haystack of size 100000   at index 9000     in 4.01072e-04 seconds
Value 1000000  found in haystack of size 100000   at index -1       in 4.63132e-03 seconds
Value 1        found in haystack of size 1000000  at index 1        in 1.95972e-07 seconds
Value 6000     found in haystack of size 1000000  at index 6000     in 2.71331e-04 seconds
Value 9000     found in haystack of size 1000000  at index 9000     in 4.11316e-04 seconds
Value 1000000  found in haystack of size 1000000  at index -1       in 4.54682e-02 seconds
```

#### Бинарный поиск - O(logN)

```python
def binary_search(needle, haystack):
    imin, imax = 0, len(haystack)
    while True:
        if imin >= imax:
            return -1
        midpoint = (imin + imax) // 2
        if haystack[midpoint] > needle:
            imax = midpoint
        elif haystack[midpoint] < needle:
            imin = midpoint + 1
        else:
            return midpoint
```

```
Value 1        found in haystack of size 10000    at index 1        in 1.44807e-06 seconds
Value 6000     found in haystack of size 10000    at index 6000     in 1.93390e-06 seconds
Value 9000     found in haystack of size 10000    at index 9000     in 1.80470e-06 seconds
Value 1000000  found in haystack of size 10000    at index -1       in 2.39607e-06 seconds
Value 1        found in haystack of size 100000   at index 1        in 1.83531e-06 seconds
Value 6000     found in haystack of size 100000   at index 6000     in 2.68531e-06 seconds
Value 9000     found in haystack of size 100000   at index 9000     in 2.29226e-06 seconds
Value 1000000  found in haystack of size 100000   at index -1       in 2.90966e-06 seconds
Value 1        found in haystack of size 1000000  at index 1        in 2.06331e-06 seconds
Value 6000     found in haystack of size 1000000  at index 6000     in 2.57271e-06 seconds
Value 9000     found in haystack of size 1000000  at index 9000     in 2.87756e-06 seconds
Value 1000000  found in haystack of size 1000000  at index -1       in 3.44557e-06 seconds
```

Python имеет встроенный модуль [bisect](https://docs.python.org/3/library/bisect.html#module-bisect), позволяющий работать с отсортированными массивами. 

### Список VS кортеж

Важное отличие кортежа от списка в том, что первый кэшируется Python'ом во время исполнения программы, поэтому нам не нужно каждый раз обращаться к ядру для резервирования блока памяти. 

#### Когда что лучше использовать

| Задача                           | Выбор  | Пояснение          |
|----------------------------------|--------|--------------------|
| Первые 20 простых чисел          | Кортеж | Данные статичны    |
| Названия языков программирования | Список | Данные дополняются |
| Возраст, вес и рост человека     | Список | Данные меняются    |
| День и место рождения человека   | Кортеж | Данные статичны    |
| Результат конкретной игры        | Кортеж | Данные статичны    |
| Результаты идущей серии игр      | Список | Данные дополняются |

#### Список как динамический массив

Динамический массив поддерживает метод `resize`, увеличивающий его емкость. Когда в список размера `N` впервые добавляется элемент, Python создает новый список. Но вместо `N + 1` выделяется `M` ячеек, где `M` > `N`. Делается это для того, чтобы иметь запас для последующих `append`'ов.

Это загадочное M равно следующему выражению:

```python
M = N + (N >> 3) + (3 if N < 9 else 6)
```

```c
new_allocated = (size_t)newsize + (newsize >> 3) + (newsize < 9 ? 3 : 6);
```

| N   | 0   | 1-4 | 5-8 | 9-16 | 17-25 | 26-36 | 37-46 | 47-58 | 59-72 | 73-88 |
|-----|-----|-----|-----|------|------|-------|-------|-------|-------|-------|
| M   | 0   | 4   | 8   | 16   | 25   | 36    | 46    | 58    | 72    | 88    |

```ipython
In [20]: %memit [i*i for i in range(100_000)]
peak memory: 59.20 MiB, increment: 3.81 MiB

In [21]: %%memit
    ...: l = []
    ...: for i in range(100_000):
    ...:     l.append(i * 2)
    ...: 
peak memory: 59.84 MiB, increment: 2.26 MiB

In [22]: %timeit [i*i for i in range(100_000)]
4.55 ms ± 17.3 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)

In [23]: %%timeit
    ...: l = []
    ...: for i in range(100_000):
    ...:     l.append(i * 2)
    ...: 
6.28 ms ± 5.16 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```

Второй вариант работает медленнее, потому что каждый раз Python должен заново выделять память.

#### Кортеж как статический массив

Кортеж не поддерживает добавление, однако можно соединить два кортежа, сформировав новый. Операция аналогична `resize` у списка, но не выделяет дополнительную память. Однако если `append` у списка работает за `O(1)`, то такой способ у кортежей работает за `O(N)`, ибо нужно каждый раз копировать их.

Когда переменная не используется, Python освобождает память и передает ее операционной системе для использования в других программах или для других переменных. Но для кортежей размера 1-20 это не работает - Python закэширует блок памяти, поэтому при создании нового кортежа такого же размера не нужно тратить время на поиск памяти.

```ipython
In [26]: %timeit l = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
40.8 ns ± 0.0954 ns per loop (mean ± std. dev. of 7 runs, 10,000,000 loops each)

In [27]: %timeit t = (0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
9.18 ns ± 0.00963 ns per loop (mean ± std. dev. of 7 runs, 100,000,000 loops each)
```
