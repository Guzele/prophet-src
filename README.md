# Prophet test

- [Build](#build)
- [Changes](#changes)
- [Dive into code](#dive-into-code)

## Build

Собрать prophet можно так:

    docker pull stasbel/prophet
    docker run -it -u="ubuntu" -w="/home/ubuntu/" stasbel/prophet /bin/bash -l &
    git clone https://github.com/StasBel/prophet-src
    cd prophet-src
    ./build
    
При этом вывод с ошибками перенаправится в ваш терминал. Это даже можно добавить в виде билда в ide.
    
## Changes

### Description

Здесь и далее будут описываться изменения оригинального кода prophet'a и новые результаты тестирования.

### Alg

Некоторое мое видение того, что и как можно изменять на каждом этапе алгоритма:

	1. На вход подается программа и набор тестов (>0 плохих и >0 хороших). Программа - это последовательность выражений. Сначала, необходимо понять, где ошибка и что изменять.
	Задача: отсортировать выражения по подозрительности в порядке убывания.
	Решение: сортировка по компаратору: чаще запускается на плохих и реже на хороших (понимаем с помощью профилировки на питоне).
	Как улучшить: да вроде никак, хотя можно проанализировать значимость выражений и, например, считать control flow (if/while/repeat) по умолчанию более подозрительным, чем, скажем, вызов функции. В самом деле, вроде как в плохих тестах мы очень часто прыгаем не в ту ветку.
	2. Итак, есть список выражения. Берем первые сколько-то (200-5000) и изменяем, генерируя пространство патчей (взято из SPR).
	Задача: сгенерировать патчи, модифицировав одно место в коде.
	Решение: взято из статьи про SPR.
	Как улучшить: хотелось бы, чтобы мы не генерировали мусора. Хорошим решением было бы поисследовать, какие изменения чаще приводят к хорошим патчам, а какие реже. Ну и, естественно, количество и качество изменений можно улучшать.
	3. ML
	Задача: (1) обучение: получаем файлы с feature векторами (2) extract program value features and modification features from each patch, then map into a binary feature vector (3) ml: каждому патчу даем score => сортируем по score, получая некоторый validation order
	Решение: хитрым образом мапаем (описано в статье, там мы учитываем MF и PVF) + ml (градиентный спуск?)
	Как улучшить:
		(1) можно улучшить часть learning: посмотреть на crawler, обходить больше проектов, получить новые файлы para.out и посмотреть на то, как они влияют на результаты
		(2) можно посмотреть, улучшаем ли идея с разными features
		(3) мне сложно сказать насчет ml
	4. Проходимся по патчам в validation order и проверяем, проходят ли они тесты. Основная часть времени работы.
	Задача: последовательность патчей и тесты, найти хорошие
	Решение: применили патч, перекомпилировали, заново проходим все тесты
	Как улучшить: казалось бы, здесь мы делаем очень много ненужной работы. Допустим у php 7000 тестов. Наврятли изменение одного выражения влияет на все 7k, но вот как это понять не совсем ясно. Скорее всего, эту часть принципиально улучшить тоже не получится.

### TODO Changes

1. [x] Исправить некорректный алгоритм подсчета числа кандидатов и схем.
2. [x] Добавить в логи вывод рага, схемы и score.
3. [ ] Docs & comments everywhere.
4. [ ] Переписать алгоритм локализации.

### Results of changes

1. Nothing (*old*, *cmd test 3 1*):

	    ==============STAT==============
		test | candidates | schemas | test_eval | fixies
		php-2adf58: 51306 40152 23228 1
		php-5a8c917: 36940 27437 35137 1
		php-8ba00176: 87574 69387 56213 1
		==============METRICS==============
		average number of fixies over tests: 1.0
		average rank of fixies over tests: 1007.33333333
		average ratio of first fix: 0.0184925680546
		average ratio of first schema: 0.000386309594703

## Dive into code

### Observations

1. Инкрементальная компиляция выполняется только для php, причем она тупо захардкожена в код prophet'a.
2. 
   С локализацией ошибок все интересно: сначала мы запускаем тестирование на всех плохих тестах. Потом берем минимум и максимум из номеров плохих,
   формируя некий отрезок из номеров, которые нам интересны, а потом увеличиваем этот отрезок в обе стороны на 200. Хорошие тесты
   берем только с номерами, попавшими в новый отрезок. Зачем так сделано - не ясно.

	Далее, для каждого места в коде, мы формируем четверку (-score, beforeend\_cnt, loc, pid), где score=neg*1000000 - pos, где neg - количество запусков на плохих тестат, а pos - на хороших, beforeend\_cnt - предположительно, минимальное время в тиках до завершения работы начиная с этого выражения, loc - место в коде, включающее в себя файл и номер строки/столбца, pid - грубо говоря, номер теста на котором достигнуто минимальное beforeend_cnt. Далее мы заводим две очереди с приоритетом, в первую пихаем 4980 максимальных четверок по всем выражениям из плохих тестов, во вторую - 20 максимальных во всем выражениям из плохих, но только из bugged\_files, если указано. Далее достаем по очереди все из первой очереди, потом все из второй, проверяя что не попали в одинаковые loc'и.

	Весь этот способ вызывает сомнения: способ вычисления score, константы 4980, 20 и 1000000 надо как-то объяснять.
3. 
   Внутри prophet'a неправильно считается количество патчей (но есть скрипт, который делает это правильно, он и использовался для статьи). Дело в том, что
   некоторые partial instantained pathces могут иметь повторяющиеся abstract conditions, которые внутри представляются разные классами clang::Expr с одинаковыми
   внутренностями (поэтому prophet и не видит разницы, помещая все в set). Это легко исправляется. Скорее всего, эта ошибка идет еше с SPR.
   
   Исправил.
