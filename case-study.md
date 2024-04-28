# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: время выполнения программы.

Сначала сделал гипотезу о том, что асимптотика времени работы программы квадратичная: отношение количества записей к времени выполнения в секундах: 100000/115 750000/61 50000/26, 25000/6). Подтвердил эту гипотезу с помощью теста rspec-benchmark. 
В таком случае для полного объема понадобится 4.7 дней.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за *время, которое у вас получилось*

Вот как я построил `feedback_loop`: профилирование - изменение кода - тестирование – бенчмаркинг – откат при отсутствии разницы от оптимизации/сохранение результатов

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался rbspy

Вот какие проблемы удалось найти и решить

### Находка №1
- rbspy показал `83.55    83.55  block (2 levels) in work - task-1.rb:101`: вызов `sessions.filter {}` на каждой итерации по `users.each`;
- перед `users.each` сгруппировал `sessions_by_user = sessions.group_by { |session| session['user_id'] }`, в `each` использовал как `sessions_by_user[user['id']] || []`
- время выполнения программы для 100к входных данных сократилось с 115с до 4с
- исправленная проблема перестала быть главной точкой роста, rbspy показал, что теперь это `98.49   100.00  block in work - task-1.rb:56`

### Находка №2
- stackprof cli показал `7126  (99.4%)          11   (0.2%)     Array#each`, он вызывается несколько раз, наибольшее `6504  (   91.3%)  Object#work]`. Поскольку rbspy указывал на `task-1.rb:56`, что является `end` `each` блока, пробую вынести этот`each` в отдельный метод `parse_file`и подтвердить гипотезу, которая и подтверждается: `5765  (99.8%)        5525  (95.7%)     Object#parse_file`. Теперь нужно разобраться, какая именно операция в этом блоке `each` требует оптимизации, `stackprof stackprof.dump --method Object#parse_file` показывает, что это заполнение массива сессий: `5261   (93.2%) /  5133  (90.9%)  |    52  |     sessions = sessions + [parse_session(line)] if cols[0] == 'session'`.
- вместо `sessions = sessions + [parse_session(line)] if cols[0] == 'session'` использую `sessions << parse_session(line) if cols[0] == 'session'`. аналогично для `users`
- время выполнения программы для 500к входных данных сократилось с 100с до 13с
- исправленная проблема перестала быть главной точкой роста, stackprof cli показал, что теперь это `558 (100.0%)         202  (36.2%)     Object#work`

### Находка №3
- `ruby-prof` в режиме `Graph` показывает, что точкой роста является `25.55%	25.55%	8.23	8.23	0.00	0.00	154066	Array#+` в `8.23	8.23	0.00	0.00	154066/154066	Array#each`. под это описания подходит 108 строка.
- вместо `users_objects = users_objects + [user_object]` используем `users_objects << [user_object]`
- время выполнения программы для 500к входных данных сократилось с 12с до с 6c
- исправленная проблема перестала быть главной точкой роста, ruby prof показал, что теперь это `66.16%	26.52%	13.47	5.40	0.00	8.07	500000	Array#all?`

### Находка №3
- `ruby-prof` в режиме `Graph` показывает, что точкой роста является `25.55%	25.55%	8.23	8.23	0.00	0.00	154066	Array#+` в `8.23	8.23	0.00	0.00	154066/154066	Array#each`. под это описания подходит 108 строка.
- вместо `users_objects = users_objects + [user_object]` используем `users_objects << [user_object]`
- время выполнения программы для 500к входных данных сократилось с 12с до с 6c
- исправленная проблема перестала быть главной точкой роста, ruby prof показал, что теперь это `66.16%	26.52%	13.47	5.40	0.00	8.07	500000	Array#all?`

### Находка №4
- `ruby-prof` в режиме `Graph` показывает, что точкой роста является `8.03	5.25	0.00	2.78	42580848/42580848	BasicObject#!=	85` в `66.16%	26.52%	13.47	5.40	0.00	8.07	500000	Array#all?`.
- вместо `if uniqueBrowsers.all? { |b| b != browser }` используем `unless uniqueBrowsers.include?(browser)`
- время выполнения программы для 500к входных данных сократилось с 6с до с 5c
- исправленная проблема перестала быть главной точкой роста, ruby prof показал, что теперь это `66.16%	26.52%	13.47	5.40	0.00	8.07	500000	Array#all?`

### Находка №5
- `ruby-prof` в режиме `Graph` показывает, что точкой роста является `2.65	0.81	0.00	1.84	846263/846265	Array#map	120` в `94.64%	22.99%	7.22	1.75	0.00	5.47	11	Array#each`. Больше всего вызовов из `Object#collect_stats_from_users`
- объединяем все блоки вызова `collect_stats_from_users` в один
- время выполнения программы для 1кк входных данных сократилось с 12с до с 10c
- исправленная проблема перестала быть главной точкой роста, ruby prof показал, что теперь это `27.07%	16.32%	3.99	2.41	0.00	1.58	846230	<Class::Date>#parse`

### Находка №5
- `ruby-prof` в режиме `Graph` показывает, что точкой роста является `27.07%	16.32%	3.99	2.41	0.00	1.58	846230	<Class::Date>#parse`, это строка `user.sessions.map{|s| s['date']}.map {|d| Date.parse(d)}.sort.reverse.map { |d| d.iso8601 }`
- вместо `Date.parse(d)` используем `Date.strptime(d, '%Y-%m-%d')` (заранее известен формат). Даты часто повторяются, используем мемоизацию для уже распаршенных дат.
- время выполнения программы для 1кк входных данных сократилось с 10с до с 7.7c
- исправленная проблема перестала быть главной точкой роста.

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с 4.7 дней до 13 секунд и уложиться в заданный бюджет.

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы добавил два теста: прогон на полных данных до 15 секунд, проверка на линейную асимптотику