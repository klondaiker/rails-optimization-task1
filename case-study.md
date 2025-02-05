# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: Программа должна корректно обработать `3_250_940` строк файла `data_large.txt` не больше `30 секунд`

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за 2 часа

Вот как я построил `feedback_loop`: 

- модифицировал программу, для облегчения тестирования: передача произвольных файлов, отключение GC
- написал тест производительности
- попытался найти ассимптотику (с помощью benchmark и отключенным gc) \
Кол-во строк | время работы (cек) \
10_000 | 0.809 \
20_000 | 2.940 \
30_000 | 6.346 \
40_000 | 11.200 \
50_000 | 18.053 \
80_000 | 47.421 \
100_000 | 78.067

Асимптотика степенная O(N^M), конкретнее степень сказать затруднительно

- построил отчёты из `ruby-prof`, `stackprof`, `rbspy`
- по каждому отчету понял какая главная точка роста
- внес необходимые правки для прохождения теста

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался `ruby-prof`, `stackprof`, `rbspy`

Вот какие проблемы удалось найти и решить

### Ваша находка №1
- ruby-prof#flat показал что 77% времени тратится на Array#select
- select использовался для поиска сессии для каждого юзера в генерации объектов пользователей \
(асимптоматика в худшем случае была O(N*M), где N - кол-во сессии, M - кол-во юзеров), 
список пользователей превратил в хеш, где в качестве ключа использую user_id, а в качестве значения стал объект \
пользователя, асимпоматика превратилась в просто O(N)
- 100_000 строк обрабатывается теперь за 1.307 сек, но цель в `3_250_940` строк за меньше 30 секунды пока не достигнута

### Ваша находка №2
- ruby-prof#graph показал что 24% времени тратится на Array#all? и 21% на BasicObject#!= 
- Array#all? используется для подсчёта количества уникальных браузеров, изменил Array на Set 
- 100_000 строк обрабатывается меньше 1 секунды
- 500_000 строк обрабатывается за 12.5 сек, но цель в `3_250_940` за меньше 30 секунды еще пока не достигнута

### Ваша находка №3
- ruby-prof#callstack показал что 71.83% времени тратится на Object#collect_stats_from_users
- Что сделано:
1) Сократил количество вызовов  Object#collect_stats_from_users до одного
2) Сократил лишние  итерации
3) Date#parse - медленная операция и она используется для преобразования в iso8601, но даты в файле \
и так в iso8601, поэтому смысла нет в Date#parse
- 500_000 строк обрабатывается за 4.04 сек, но цель в `3_250_940` за меньше 30 секунды еще пока не достигнута

### Ваша находка №4
- ruby-prof#flat показал что 24.29 тратится на String#split
- убрал лишние вызовы split
- 500_000 строк обрабатывается за 2.5 сек
- 3_250_940 строк обрабаывается за 63 сек - ищем дальше

## Ваша находка №5
- Подключил GC
- 3_250_940 строк обрабатывается за 24 сек - бинго!
- Как ни парадаксально, но включение GC уменьшило время обработки \
тк при выключенном GC на моем компьютере заканчивалась память и все уходило в своп
- Вывод: при оптимизации производительности следить за окружением

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы обработки с больше часа до 25 сек и уложиться в заданный бюджет.

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы я написал performance-тест - `performance_spec.rb`

