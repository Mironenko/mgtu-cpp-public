## Многопоточность

### Справочные материалы
* thread
	* http://en.cppreference.com/w/cpp/thread/thread
	* http://en.cppreference.com/w/cpp/thread/thread/thread
	* 
* this_thread
	* http://en.cppreference.com/w/cpp/thread/sleep_for
* chrono
	* http://en.cppreference.com/w/cpp/chrono/duration
* mutex
	* http://en.cppreference.com/w/cpp/thread/mutex
* lock_guard
	* http://en.cppreference.com/w/cpp/thread/lock_guard
* unique_lock
	* http://en.cppreference.com/w/cpp/thread/unique_lock
* conditional_variable
	* http://en.cppreference.com/w/cpp/thread/condition_variable
* misc
	* https://habrahabr.ru/post/182610/
	* http://scrutator.me/post/2012/04/04/parallel-world-p1.aspx
	* http://xydan.livejournal.com/8595.html

### Threads basics (2)

* **(0,5 -- MT-NOSYNC)**  Реализовать выполнение двух функций (f1, f2) в отдельных потоках:
	* потоки выполнения f1 и f2 запускаются одновременно;
	* f1 выполняет вывод нечетных чисел в промежутке от 1 до 100 c задержкой между выводом в 0.1 с;
	* f2 выполняет вывод четных чисел в промежутке от 1 до 100 c задержкой между выводом в 0.1 с;
* **(0,5 -- MT-SYNC)**  Модифицировать задачу MT-NOSYNC таким образом, чтобы каждая из функций выводила как минимум 2 числа подряд (с сохранением задержек между выводом);
* **(0,5 -- MT-WAIT)**  Модифицировать задачу MT-SYNC таким образом, чтобы вывод четных чисел начинался не раньше того момента, когда будут выведены 50 нечетных чисел.
* **(0,2 -- DEADLOCK-M)**  Продемонстрировать случай взаимной блокировки потоков в ожидании мьютекса.
* **(0,3 -- DEADLOCK-T)**  Продемонстровать случай взаимной блокировки потоков при ожидании их завершения (циклический join).

### Background commands
Реализовать командный процессор, поддерживающий выполнение следующих функций:
* echo -- синхронная функция, печатающая на новой строке переданный ей текст без задержек
```
> echo Hello
 Hello
> echo aaaa
 aaaa
```
* **(0,5 -- ALARM)** alarm -- функция установки "таймера", выводящего сообщение о завершении по истечении указанного промежутка времени:
```
> alarm 10
alarm 32131 created
> echo a
a
>
// 10 seconds passed
Alarm!!!
```
* **(1 -- C-ALARM)** calarm, start, cancel -- функция установки, запуска и сброса таймера, выводящего сообщение о завершении по истечении указанного промежутка времени:
```
> calarm 10
calarm 89273 created
> start 89273
> echo a
a
>
// 10 seconds passed
Alarm!!!
> calarm 10
calarm 53256 created
> start 53256
> echo a
a
> cancel 53256
calarm 53256 cancelled
// 10 seconds passed -- nothing
```
* **(1 -- FIND)** find, cancel -- функция поиска в файле подстроки
```
> find file1 aaaaa
file1 is being processed 76278
>
// file was read till the end
(76278) aaaaaa was found 9 times
>
```
* **(1 -- BIND-ALARM)** bind-сalarm -- функция отложенного выполнения команды по истечении таймера
```
> calarm 10
calarm 53256 created
> bind-calarm 53256 echo aaaaaa
> start 53256
> echo a
a
>
// 10 seconds passed
Alarm!!!
aaaaaa
```
* **(1 -- BIND-FIND)** bind-find -- функция отложенного выполнения команды по обнаружении подстроки в файле:
```
> find file1 aaaaa
file1 is being processed 76278
> bind-find 76278 echo bbbbbb
bbbbbb
bbbbbb
bbbbbb
bbbbbb
// file was read till the end
(76278) aaaaaa was found 4 times
>
```