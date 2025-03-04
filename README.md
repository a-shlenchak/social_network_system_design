# System Design социальной сети для курса по System Design

---

## Functional Requirements

- посты
  - публикация:
    - пост содержит название, текст, фотографии и локацию
    - локация содержит название страны, город, координаты
    - при добавлении поста поиск локации по названию, либо возможность добавить локацию
  - чтение:
    - получение постов для пользователя (с сортировкой по дате создания в порядке убывания)
    - получение постов для локации (с сортировкой по дате создания в порядке убывания)
    - получение поста с комментариями и оценками
  - оценка постов
  - комментирование поста:
    - добавление комментария (только текст)
    - получение комментариев к посту с сортировкой по дате создания в порядке убывания
- подписка:
  - возможность подписаться на пользователя
  - возможность отписаться от пользователя
  - посты пользователя, на которого подписан, добавляются в собственную ленту
- локации:
  - поиск по названию
  - список локаций, отсортированный по убыванию количества постов, относящихся к локации
  - лента постов, относящихся к локации


---

## NFR

### Данные

- лимиты:
  - посты:
    - наименование - 50 символов
    - текст - 3500-4000 символов
    - фотограции - максимум 7 фото на пост. Максимальный размер 3Мб, средний размер 2Мб. После загрузки сжимаются и пользователь получает сжатые фото. Средний размер сжатого фото - 150-200кб
  - локации:
    - название страны - 70 символов
    - город - 50 символов
    - координаты - по 10 символов, итого 20 символов
  - комметарии:
    - текст - максимум 200 символов
- данные храним всегда

### Аудитория

DAU - 10 000 000

Регион - СНГ

Сезонность - присутствует. С мая по сентябрь рост трафика в 2,5-3 раза. С середины декабря по конец января рост трафика в 2раза.

### Поведение пользователей

- добавление 1 поста в неделю
- проверка ленты 2 раза в день (загрузка 10 постов за раз)
- просмотр ленты других пользователей 1 раз в день
- поиск локаций 2 раза в день
- просмотр ленты постов о локациях 2 раза в день
- чтение 10 постов в день (суммарно из своей ленты, ленты других пользователей, о локации)
- чтение 20 комментариев в день
- добавление 3 комментариев в день
- оценка 5 постов в день
- подписка/отписка 5 раз в день

### Ограничения для пользователей

- добавление максимум 5 постов в день
- чтение постов - нет ограничений
- максимальное количество подписок - 1 000
- максимальное количество подписчиков у одного пользователя - 100 000
- максимальное количество оценок постов - 50 в день
- максимальное количество комментариев постов - 50 в день

### Тайминги

- создание поста - 1000мс
- загрузка фотографии - 1500мс
- получение постов ленты - 2000мс
- поиск локации - 1500мс
- добавление комментария - 1000мс
- оценка поста - 700мс
- подписка/отписка - 700мс

---

## Оценка нагрузки

### Посты

###### Write: 

Отдельно оценим нагрузку для поста без загрузки фото и загрузку фото, тк данные операции будут выполняться 2мя разными запросами.
Учтем, что 1 пост в неделю - это ~ 0.15 в день.

```
RPS (write) для поста: 10 000 000 * 0.15 / 100 000 = 15
```

Тк основной объем информации для поста (без учета картинок) - это текст поста, остальными данными можем пренебречь. Текст будем хранить в кодировке utf-8, размер кириллического символа - 3 байта.
Тогда размер поста - 3 * 4 000 = 12 000 b.

```
Traffic (write) для поста: 12 000 * 15 = 180 000 b/s = 180 kb/s или 0.18 Mb/s
```

С учетом того, что максимальное количество фотографий - 7 на пост:

```
RPS (write) для фотографий: 15 * 7 = 105
```

Учитывая средний размер картинки (2 Mb) и RPS (105), получаем

```
Traffic (write) для фотографий: 105 * 2 = 210 Mb/s
```

###### Read:

Отдельно оценим нагрузку по чтению списка постов и самих постов. Список постов будет содержать заголовок + главное фото + информацию по количеству оценок и комментариев поста для каждого поста в списке.


Количество просмотров списка (ленты) постов складывается из 3х слагаемых: просмотр своей ленты (2 раза в день), просмотр ленты других пользователей (1 раз в день) и просмотр ленты постов о локации (2 раза в день).
Итого 5 просмотров различных лент в день 

```
RPS (read) для постов в ленте: 10 000 000 * 5 / 100 000 = 500
```

Такое же количество запросов будет и для получения главного фото поста:

```
RPS (read) для фото постов в ленте: 10 000 000 * 5 / 100 000 = 500
```

Для ленты загружается 10 постов за раз. С учетом сортировки постов от новых к старым, дополнительной подгрузкой следующей "страницы" можем пренебречь.
Всего, таким образом, пользователь в различных лентах будет загружать до 50 постов в день.
Размер поста в ленте - это, по сути, размер заголовка: 3 * 50 = 150 b

```
Traffic (read) для постов в ленте: 500 * 50 * 150 = 3 750 000 b/s = 3 750 kb/s или 3.75 Mb/s
```

Т.к. средний размер возвращаемой картинки - 200 kb, то

```
Traffic (read) для фото постов в ленте: 500 * 50 * 200 = 5 000 000 kb/s = 5 000 Mb/s или 5 Gb/s
```

При чтении поста пользователь получает наименование, текст поста и фотографии. Посчитаем отдельно нагрузку для получения поста и фотографий (это будут разные запросы).

```
RPS (read) для просмотра поста: 10 000 000 * 10 / 100 000 = 1 000
```

Текст поста заметно больше его наименования, поэтому для расчетов размером наименования можем пренебречь. Размер текста поста мы считали выше и он равен 12 000 b.

```
Traffic для просмотра поста: 1 000 * 12 000 = 12 000 000 b/s = 12 000 kb/s или 12 Mb/s
```

Для поста мы загружаем максимум 7 фото:

```
RPS (read) для фотографий поста: 1 000 * 7 = 7 000
```

Т.к. средний размер возвращаемой картинки - 200 kb, то

```
Traffic (read) для фотографий поста: 7 000 * 200 = 1 400 000 kb/s = 1 400 Mb/s или 1.4 Gb/s
```

### Оценки

Тк оценки будут возвращаеться с постами, то отдельного RPS (read) для них не будет.
Оценим только Write нагрузку для них

```
RPS (write) для оценок: 10 000 000 * 5 / 100 000 = 500
```

Для оценки трафика примем в качестве размера запроса на добавление оценки величину в 1 kb:

```
Traffic (write) для оценок: 500 * 1 = 500 kb/s или 0.5 Mb/s
```

### Комментарии

###### Write:

```
RPS (write): 10 000 000 * 3 / 100 000 = 300
```

Размер одного комментария - максимум 200 символов. Тогда его размер в байтах будет 3 * 200 = 600 b.

```
Traffic (write): 300 * 600 = 180 000 b/s = 180 kb/s или 0.18 Mb/s
```

###### Read:

Пусть пользователь в среднем читает 20 комментариев к постам. При этом он читает в среднем 10 постов. Тогда получим, что пользователь читает в среднем 2 комментария к посту. Комментарии будем грузить батчами по 5 штук.
Предположим, что к некоторым постам он захочет прочитать не только самые последние 5 комментариев, но и более ранние. Тогда у пользователя будет в среднем 15 загрузок веток комментариев батчами по 5 штук. Итого 75 комментариев:

```
RPS (read): 10 000 000 * 15 / 100 000 = 1 500
```

Размер одного комментария мы считали выше - 600 b:

```
Traffic (read): 1 500 * 600 = 900 000 b/s = 900 kb/s или 0.9 Mb/s
```

### Локации

###### Write:

Создание локаций происходит при создании поста - 0.15 раз в день для пользователя. Предполагается, что постепенно нагрузка на запись будет стремиться к 0.
Посчитаем начальные значения

```
RPS (write): 10 000 000 * 0.15 / 100 000 = 15 
```

Размер локации: страна + город + координаты - 3 * 70 + 3 * 50 + 20 = 360 b. Округлим до 400 b.

```
Traffic (write): 15 * 400 = 6 000 b/s = 6 kb/s
```

###### Read:

Поиск локации будет происходить в 2х местах: непосредственно в локациях, при создании поста. Тогда получаем среднее количество поисков локаций: 2 + 0.15 = 2.15

```
RPS (read): 10 000 000 * 2.15 / 100 000 = 215
```

Тк размер локации - 400 b:

```
Traffic (read): 215 * 400 = 86 000 b/s = 86 kb/s 
```

### Подписка/отписка

Так, как операции подписки/отписки предполагают только запись, будем оценивать только такой тип нагрузки

```
RPS (write): 10 000 000 * 5 / 100 000 = 500
```

Для оценки трафика примем в качестве размера запроса на подписку/отписку величину в 1 kb:

```
Traffic (write): 500 * 1 = 500 kb/s или 0.5 Mb/s
```

## Оценка дисков

### Посты

**Capacity**

0.18 * 86 400 * 365 = ~6TB

| Вид нагрузки         | HDD | SSD (sata) | SSD (nvme) |
|----------------------|-----|------------|------------|
| capacity (6TB)       | 1   | 1          | 1          |
| iops (>1500RPS)      | 16  | 2          | 1          |
| throughput (~16MB/s) | 1   | 1          | 1          |

Для постов остановимся на 16 HDD объемом по 1TB

### Фото постов

**Capacity**

210 * 86 400 * 365 = ~7PB

| Вид нагрузки          | HDD | SSD (sata) | SSD (nvme) |
|-----------------------|-----|------------|------------|
| capacity (7PB)        | 250 | 70         | 240        |
| iops (~7500RPS)       | 75  | 8          | 1          |
| throughput (~6.5GB/s) | 65  | 13         | 3          |

Для хранения горячих данных (на 2 месяца) будем использовать SSD, охлаждать будем на HDD. 
Тогда для горячих данных нам потребуется: 70 * 1/6 = 12 SSD объемом по 100TB.
Для холодных данных: 250 * 5/6 = 209 HDD объемом по 32TB.

### Оценки

**Capacity**

Размер записи 1 оценки в таблице - ~32 B. Посчитаем capacity по RPS:

32 * 500 * 86 400 * 365 = ~505GB

| Вид нагрузки         | HDD | SSD (sata) | SSD (nvme) |
|----------------------|-----|------------|------------|
| capacity (505GB)     | 1   | 1          | 1          |
| iops (~500RPS)       | 5   | 1          | 1          |
| throughput (~16kB/s) | 1   | 1          | 1          |

Для оценок остановимся на 1 SSD (sata) объемом 1TB

### Комментарии

**Capacity**

0.18 * 86 400 * 365 = ~6TB

| Вид нагрузки         | HDD | SSD (sata) | SSD (nvme) |
|----------------------|-----|------------|------------|
| capacity (6TB)       | 1   | 1          | 1          |
| iops (1800RPS)       | 18  | 2          | 1          |
| throughput (~16MB/s) | 1   | 1          | 1          |

Для комментариев остановимся на 18 HDD объемом по 1TB

### Локации

**Capacity**

6 * 86 400 * 365 = ~190GB

| Вид нагрузки          | HDD | SSD (sata) | SSD (nvme) |
|-----------------------|-----|------------|------------|
| capacity (190GB)      | 1   | 1          | 1          |
| iops (230RPS)         | 3   | 1          | 1          |
| throughput (~100kB/s) | 1   | 1          | 1          |

Для локаций остановимся на 3 HDD объемом по 1TB

### Подписки

**Capacity**

Размер записи 1 подписки в таблице - ~50 B. Посчитаем capacity по RPS:

50 * 500 * 86 400 * 365 = ~800GB

| Вид нагрузки         | HDD | SSD (sata) | SSD (nvme) |
|----------------------|-----|------------|------------|
| capacity (800GB)     | 1   | 1          | 1          |
| iops (500RPS)        | 5   | 1          | 1          |
| throughput (~25kB/s) | 1   | 1          | 1          |

Для подписок остановимся на 1 SSD (sata) объемом 1TB

## Оценка постов

### Посты

* Master-slave, async, RF 2
* Sharding: key-based by user_id 
* Hosts = 16 / 2 = 8
* Hosts_with_replication = 8 * 2 = 16 

### Фото постов

* Master-slave, async, RF 2
* Hosts (SSD) = 12 / 6 = 2
* Hosts_with_replication (SSD) = 2 * 2 = 4
* Hosts (HDD) = 209 / 7 = 30
* Hosts_with_replication (HDD) = 30 * 2 = 60

### Оценки

* Master-slave, async, RF 2
* Hosts = 1 / 1 = 1
* Hosts_with_replication = 1 * 2 = 2

### Комментарии

* Master-slave, async, RF 2
* Sharding: key-based by post_id
* Hosts = 18 / 3 = 6
* Hosts_with_replication = 6 * 2 = 12

### Локации

* Master-slave, async, RF 2
* Hosts = 3 / 3 = 1
* Hosts_with_replication = 1 * 2 = 2

### Подписки

* Master-slave, async, RF 2
* Hosts = 1 / 1 = 1
* Hosts_with_replication = 1 * 2 = 2
