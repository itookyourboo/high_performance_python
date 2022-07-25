# Профилирование

### Декоратор измерения времени работы

```python
import time
from functools import wraps

def timefn(fn):
    @wraps(fn)
    def measure_time(*args, **kwargs):
        t1 = time.time()
        result = fn(*args, **kwargs)
        t2 = time.time()
        print(f"@timefn: {fn.__name__} took {t2 - t1} seconds")
        return result
    return measure_time


@timefn
def calculate_z_serial_purepython(maxiter, zs, cs):
    ...
```

### timeit, командная строка

```shell
$ python -m timeit -n 5 -r 1 -s "import julia1" \
"julia1.calc_pure_python(desired_width=1000, max_iterations=300)"
```

```
5 loops, best of 1: 5.17 sec per loop
```

| Опция              | Пояснение                                                        |
|--------------------|------------------------------------------------------------------|
| -n 5               | Количество запусков кода                                         |
| -r 5               | Количество повторений эксперимента (1 r = 5 n)                   |
| -s "import julia1" | Настройка эксперимента перед исполнением кода. Вызывается 1 раз. |

По умолчанию `-n 10 -r 5`

### timeit, IPython

В интерактивном режиме удобно пользоваться magic-командой %timeit

```shell
$ ipython
```

```ipython
In [1]: import julia1

In [2]: %timeit julia1.calc_pure_python(desired_width=1000, max_iterations=300)
```

```
5.77 s ± 406 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

`timeit.py` использует минимальное значение<br>
`%timeit` использует среднее время и стандартное отклонение

### Unix-утилита time

```shell
$ /usr/bin/time -p python julia1_nopil.py 
```
```
Length of x: 1000
Total elements: 1000000
calculate_z_serial_purepython took 5.151060104370117 seconds
real 5.45
user 5.42
sys 0.02
```

Используется `/usr/bin/time` вместо `time` используется намеренно, чтобы получить именно системную утилиту, а не упрощенную версию из командной оболочки.

Флаг `-p` позволяет получить три значения:

- `real` полное затраченное время
- `user` время, затраченное ЦПУ на исполнение задачи вне ядра
- `sys` время на исполнение функций ядра

Разница между `real` и `user` может указать на `%IO` - ожидание ввода/вывода, а также насколько система загружена другими задачами.

`time` - очень полезная утилита, потому что она не привязана к Python. Хорошо подходит для коротких по времени исполнения программ. 

Можно получить гораздо больше информации, добавив флаг `-v` или `--verbose`:

```shell
$ /usr/bin/time -v python julia1_nopil.py
```
```
Length of x: 1000
Total elements: 1000000
calculate_z_serial_purepython took 5.176875591278076 seconds
        Command being timed: "python julia1_nopil.py"
        User time (seconds): 5.43
        System time (seconds): 0.03
        Percent of CPU this job got: 99%
        Elapsed (wall clock) time (h:mm:ss or m:ss): 0:05.47
        Average shared text size (kbytes): 0
        Average unshared data size (kbytes): 0
        Average stack size (kbytes): 0
        Average total size (kbytes): 0
        Maximum resident set size (kbytes): 98432
        Average resident set size (kbytes): 0
        Major (requiring I/O) page faults: 0
        Minor (reclaiming a frame) page faults: 23464
        Voluntary context switches: 1
        Involuntary context switches: 22
        Swaps: 0
        File system inputs: 0
        File system outputs: 0
        Socket messages sent: 0
        Socket messages received: 0
        Signals delivered: 0
        Page size (bytes): 4096
        Exit status: 0
```

Наверное, самым полезным показателем является `Major (requiring I/O) page faults`. Он показывает, нужно ли загружать страницы данных с диска из-за того, что данные уже не хранятся в оперативной памяти. Это сильно снижает скорость работы.

### Модуль cProfile

В стандартной библиотеке есть два профилировщика: `cProfile` и `profile.profile`.

Первый написан на C и измеряет время работы в виртуальной машине CPython.<br>
Второй написан на чистом Python, поэтому он работает медленнее.

```shell
$ python -m cProfile -s cumulative julia1_nopil.py 
```
```
Length of x: 1000
Total elements: 1000000
calculate_z_serial_purepython took 10.527417659759521 seconds
         36221995 function calls in 11.142 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000   11.142   11.142 {built-in method builtins.exec}
        1    0.020    0.020   11.142   11.142 julia1_nopil.py:1(<module>)
        1    0.479    0.479   11.122   11.122 julia1_nopil.py:23(calc_pure_python)
        1    7.942    7.942   10.527   10.527 julia1_nopil.py:9(calculate_z_serial_purepython)
 34219980    2.585    0.000    2.585    0.000 {built-in method builtins.abs}
  2002000    0.109    0.000    0.109    0.000 {method 'append' of 'list' objects}
        1    0.007    0.007    0.007    0.007 {built-in method builtins.sum}
        3    0.000    0.000    0.000    0.000 {built-in method builtins.print}
        2    0.000    0.000    0.000    0.000 {built-in method time.time}
        4    0.000    0.000    0.000    0.000 {built-in method builtins.len}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```

Сортировка функуий по совокупному времени `-s cumulative` позволяет увидеть самые медленные части кода.

Код исполняется на 5 секунд дольше. Это плата за использование профилировщика.

Интересно, что 2.5 секунды тратится на 34219980 вызовов функции abs.

#### pstats

Чтобы лучше разобраться в результате работы cProfile, можно записать его в stat-файл, а затем проанализировать в Python:

```shell
$ python -m cProfile -o profile.stats julia1.py
```

```python
import pstats
p = pstats.Stats('profile.stats')
p.sort_stats('cumulative')
p.print_stats()
```
```
Tue Jul 26 00:28:02 2022    profile.stats

         36221995 function calls in 11.090 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000   11.090   11.090 {built-in method builtins.exec}
        1    0.023    0.023   11.090   11.090 julia1_nopil.py:1(<module>)
        1    0.415    0.415   11.066   11.066 julia1_nopil.py:23(calc_pure_python)
        1    7.901    7.901   10.551   10.551 julia1_nopil.py:9(calculate_z_serial_purepython)
 34219980    2.651    0.000    2.651    0.000 {built-in method builtins.abs}
  2002000    0.095    0.000    0.095    0.000 {method 'append' of 'list' objects}
        1    0.005    0.005    0.005    0.005 {built-in method builtins.sum}
        3    0.000    0.000    0.000    0.000 {built-in method builtins.print}
        2    0.000    0.000    0.000    0.000 {built-in method time.time}
        4    0.000    0.000    0.000    0.000 {built-in method builtins.len}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```

Получаем информацию о том, откуда были вызваны функции:

```python
p.print_callers()
```
```
   Ordered by: cumulative time

Function                                          was called by...
                                                      ncalls  tottime  cumtime
{built-in method builtins.exec}                   <- 
julia1_nopil.py:1(<module>)                       <-       1    0.023   11.090  {built-in method builtins.exec}
julia1_nopil.py:23(calc_pure_python)              <-       1    0.415   11.066  julia1_nopil.py:1(<module>)
julia1_nopil.py:9(calculate_z_serial_purepython)  <-       1    7.901   10.551  julia1_nopil.py:23(calc_pure_python)
{built-in method builtins.abs}                    <- 34219980    2.651    2.651  julia1_nopil.py:9(calculate_z_serial_purepython)
{method 'append' of 'list' objects}               <- 2002000    0.095    0.095  julia1_nopil.py:23(calc_pure_python)
{built-in method builtins.sum}                    <-       1    0.005    0.005  julia1_nopil.py:23(calc_pure_python)
{built-in method builtins.print}                  <-       3    0.000    0.000  julia1_nopil.py:23(calc_pure_python)
{built-in method time.time}                       <-       2    0.000    0.000  julia1_nopil.py:23(calc_pure_python)
{built-in method builtins.len}                    <-       2    0.000    0.000  julia1_nopil.py:9(calculate_z_serial_purepython)
                                                           2    0.000    0.000  julia1_nopil.py:23(calc_pure_python)
{method 'disable' of '_lsprof.Profiler' objects}  <- 
```

Кто что вызывал:

```python
p.print_callees()
```
```
   Ordered by: cumulative time

Function                                          called...
                                                      ncalls  tottime  cumtime
{built-in method builtins.exec}                   ->       1    0.023   11.090  julia1_nopil.py:1(<module>)
julia1_nopil.py:1(<module>)                       ->       1    0.415   11.066  julia1_nopil.py:23(calc_pure_python)
julia1_nopil.py:23(calc_pure_python)              ->       1    7.901   10.551  julia1_nopil.py:9(calculate_z_serial_purepython)
                                                           2    0.000    0.000  {built-in method builtins.len}
                                                           3    0.000    0.000  {built-in method builtins.print}
                                                           1    0.005    0.005  {built-in method builtins.sum}
                                                           2    0.000    0.000  {built-in method time.time}
                                                     2002000    0.095    0.095  {method 'append' of 'list' objects}
julia1_nopil.py:9(calculate_z_serial_purepython)  -> 34219980    2.651    2.651  {built-in method builtins.abs}
                                                           2    0.000    0.000  {built-in method builtins.len}
```

Не самый информативный инструмент, но позволяет быстро найти узкие места.
