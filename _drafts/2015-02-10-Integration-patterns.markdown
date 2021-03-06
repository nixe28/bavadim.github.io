---
layout: post
title: "Про системную интеграцию и обмен сообщениями"
comments: true
categories: programming
---

Практика построения больших программ из более мелких подсистем вошла в обиход разработчиков масштабируемых систем далеко не вчера. При понимании определенных правил проектирования реализация таких систем становится делом увлекательным и позновательным. С другой стороны задача интеграции какой-нибудь SAP с 1С может и не быть сущим адом для того, кто владеет матчастью. В этой заметке я постараюсь кратко и системно изложить принципы и свой опыт по теме. Будут примеры, но, как всегда, статья не о конкретном решении, а о технологии и идее в целом.

## Способы интеграции

Системная интеграция достаточно общий термин, под которым можно понимать как технологию объединения подсистем в одну систему, так и различные обсуждения бюджетов с заказчиком в бане. В этой заметке речь пойдет о технической стороне вопроса. Дадим такое определение:

**Интеграция систем** - это процесс создания условий для передачи информации из одних систем в другие с целью объединения систем в одну.

Интеграцию обычно сопроваждает множество проблем. Кроме обычных вопросов баланса между масштабируемостью, производительностью, простотой системы и стоимостью, приходится иметь дело со специфическими проблемами. Зачастую мы можем весьма ограничено менять существующие системы или не можем делать этого вовсе. Часто инегрируемые системы стары и плохо спроектированы, а доступ к документации и поддержке стоит немалых денег (привет, энтерпрайз). Но все эти проблемы обычно разрешимы и не сильно досождают по окончании интеграции. Настаящая головная боль - это сильная связанность подсистем. Сильная связанность проявляет себя когда вдруг  при обновлении формата фйлов в системе ECM перестают работать CRM и трекер задач. Или, например, когда заменить не удовлетваряющий вас трекер задач на что-то стоящее невозможно из-за того, что его интеграция с CI, CRM и биллингом завязана на схему БД. Сильно связанная система - плохая и дорогая штука, являющаяся головной болью для пользователей и людей, вовлеченных в обслуживание системы, которых у таких систем обычно много.

Итак, при проектировании интеграционного решения всегда приходится искать баланс между следующими свойствами:

- связанность
- масштабируемость
- производительность
- простота
- цена

К сожалению не все эти свойства можно численно измерять, а тем более посчитать численно качество интеграции. Но мы все же можем "на глаз" оценить качество того или иного типа интеграции в применении к нашей задаче. Всего существует четыре типа интеграций:

- передача файла 
- общая БД 
- удаленный вызов процедуры
- обмен сообщениями

Первые три способа просты и построены на наиболее универсальных механизмах IPC (Inter process communication). Обмен сообщений более сложная и продуманая концепция, спроектированая для построения слабосвязанных систем. Стоит сказать, что обмен сообщениями тоже стал стандартным методом IPC (например DBus в Linux).

Ниже дадим краткую характеристику каждому типу интеграции и детально остановимся на обмене сообщениями.

### Передача файла

Файлы позволяют добиться временной независимости интегрируемых систем (асинхронной обработки), ведь приложению-отправителю достаточно создать файл и совсем не обязательно ждать пока его обработает приложение-получатель. И наоборот приложению-получателю не обязательно быть доступным в момент создания файла. Это свойство можно использовать, чтобы уменьшить связанность и увеличить масштабируемость. 

Передача файла обладает низкой производительностью, так как ввод-вывод на диск операция мягко скажем не быстрая. Тем не мение это самый простой на первый взгляд способ интеграции. Большинство приложений умеют передовать информацию через файлы без каких-либо доработок. Но нужно учитывать, что файлы имеют определенный формат, с которым все интегрируемые приложения должны уметь работать. Поэтому нужно хорошо обдумать вопрос является ли для конкретной задачи интеграция через передачу фалов самым простым и дешевым способом.

### Общая БД

Особенностью этой схемы является то, что она имеет смысл для обмена хорошо структурированными данными. Как и передача файлов, общаяя БД позволяет выполнить асинхронную обработку, но при этом обеспечивает высокую пропускную способность. Другим отличием от передачи файлов является большая степень связанности интегрируемых систем. Это происходит потому, что формат файлов в отличии от схемы данных меняется реже и чаще рассматривается разработчиками как внешний интерфейс. Я встречал несколько более-менее удачных решений интеграции через БД, но во всех них таблицы, служащие для интеграции строго отделяли от данных приложения. С масштабируемостью у этого типа, как и у всего что связано с реляционными СУБД, большая проблема.

### Удаленный вызов процедур

Самый неоднозначный пункт. Сюда можно отнести Statefull EJB и RESTFul сервисы. Про первый случай я может быть напишу отдельную заметку в серии будни осинизатора. Но сейчас, все же, я буду отталкиваться от свойств RESTFul. 

Связанность примерно такая же как и у предыдущих типов. Вместо схемы БД или формата файлов клиенту нужно знать API сервера. Сервер должен быть доступен в момент обращения клиента и часто клиент должен сам запрашивать результат обработки запроса. В общем этот тип ориентирован на работу по схеме запрос-ответ, а не асинхронный режим. Поэтому я в своем личном рейтинге ставлю этот способ по простоте масштабируемости почти в самый конец (ниже только общая БД). Производительность может быть на хорошем уровне. А вот простота и цена решения сильно зависит от задачи. Если система-сервер имеет хорошее и полное API, то этот тип интеграции может быть самым простым из всех. К сожалению так бывает не всегда.

### Обмен сообщениями

Все вышеописаные способы интеграции имеют одну общую особенность - они оговаривают лишь канал обмена сообщениями (ФС, БД, API), но не оговаривают тип информации, адресацию и не вводят четкого прядка обмена информацией, остовляя большой простор для творчества. Эта свобода обычно приводит к сильносвязанной системе с последствиями, описанными в начале статьи. Кроме того многообразие делает невозможной стандартизацию и следовательно написание библиотек для интеграции. Обмен сообщениями призван решить эти проблемы.

Обмен сообщениями построен на следующих концепциях: 

- сообщение - атомарная информация, передаваемая между системами
- конечная точка - часть интегрируемой системы, создающая или получающая сообщения
- поток - очередь сообщений, гарантированнай доставка и общая доступность не обеспечиваются
- канал - общедоступный именованый поток, в который, можно записывать и из которого можно читать сообщения; доставка сообщений гарантируется
- фильтр - обработчик сообщений, который пропускает часть сообщений из входного потока в выходной
- преобразование - обработчик сообщений, изменяющий сообщения из входного потока и отправляющий в выходной
- маршрутизация - принимает сообщения из входного потока и отправляет в один из выходных потоков

На базе этих концепций можно решить любую задачу интеграции. Для решения наиболее общих задч существуют типовые приемы, узнать подробности которых можно в книге [Шаблоны интеграции корпоративных приложений](http://www.ozon.ru/context/detail/id/3083192/). Большинство паттернов реализованны в библиотеке [Apache Camel](http://camel.apache.org/). Кроме того, в этой библиотеке реализовано [множество конечных точек](http://camel.apache.org/component.html), позволяющих быстро выполнить интеграцию через распространенные интерфейсы. Поддержка каналов обычно обеспечивается специальным классом систем - системой обмена сообщениями (Message-Oriented Middleware, MoM), например [RabbitMQ](http://www.rabbitmq.com/). Сервер MoM похож на БД в том смысле что обеспечивает надежное хранение и доступ к сообщениям в каналах. Рассмотрим применение вышеописанных средств на примере одной интеграции.

## Паттерны

## Пример

Рассмотрим систему социального маркетинга. Наша система будет получать из твиттера твиты, анатированые хэштэгом нашей компании и передавать их системе лингвистического анализа, которая определет эмоциональную окраску твита (положительная или отрицательная). Результаты системы лингвистического анализа сохраняются в БД.


Глядя на пример мы можем сделать некоторые выводы. Во-первых подсистемы не связанны друг с другом. Зависят системы только от работоспособности каналов (а следовательно MoM), но не зависят от доступности друг друга. Во-вторых адреса подсистемы используют для обмена информации именованые каналы и не знают ничего друг о друге, это обстоятельство значительно уменьшает связанность и дает возможность добавлять/удалять подсистемы без изменения других подсистем. И третее наблюдение состоит в том, что сообщения могут быть различными и неплохо было бы систематизировать информацию о многообразии сообщений. Итак, далее мы рассмотрим:

- временная согласованность
- адресация и канал
- формат сообщений

## Мыслим асинхронно

Взаимодействие подсистем обычно заключается в том, что системы-поставщики поставляют сообщения, а системы-получатели обрабатывают их. Одна и та же система может быть поставщиком и получателем. Процесс отправки и приема сообщений может происходить по одной из двух схем. Первая - синхронная, при которой система-отправитель передает сообщение получателю и ждет результата обработки. Вторая схема - асинхронная. Система-отпревитель передает сообщение в канал и не ждет его обработки. Синхронная схема ообладает следующими важными свойствами: 

- позволяет отправителю сообщения отправить и забыть его
- получатель может получить его когда удобно

Эти свойства позволяют сделать подсистемы максимально независимыми друг от друга. Асинхронное интеграционное решение позволяет подсистемам работать с максимально возможной пропускной способностью (при синхронном подходе все работают со скоростью самой медленной). Кроме того асинхронная система более надежна - недоступность какй-либо подсистемы не влияет на другие компоненты. Асинхронная схема проще масштабируется - во многих случаях достаточно добавить дополнительные инстансы медленных компонентов так, чтобы они работали параллельно. Это позволит ускорить обработку почти линейно. Синхронная модель может быть построена на базе асинхронной с помощью типового решения "идентификатор корреляции" и двух очередей - входной и очереди с результатами обработки.

Понятно, что все примущества асинхронной модели дает система обмена сообщениями, которая обеспечивает:

- надежность хранения
- постоянная доступность

Многие MoM на сегоднешний день являются надежными, отказоустойчивыми и масштабируемыми решениями.

## Адресация и канал

Каналы служат для обмена сообщениями и обеспечивают косвенную адресацию. Вместо адресов систем-отправителей и систем-получателей мы указываем имена каналов. В каналы могут посылать и читать из них множество разнобразных систем. В зависимости от количества получателей сообщения, каналы делятся на следующие типы:

- точка-точка - один получатель у сообщения
- публикация-подписка - у сообщения может быть несколько получателей

Каналов может быть неограниченное число, но обычно каналы организуют по определенной схеме. Наиболее простой схемой является схема один канал - один тип данных. Эта схема наиболее проста в плане масштабирования и поддержки. Для ошибочных сообщений и сообщений, которые не удалось доставить принято выделять канал недопустимых и канал недоставленных сообщений.

## Формат сообщений

**Формат сообщения** - это спецификация структуры сообщения.

В структуру любого сообщения входит информация двух типов:

- метаданные, использующиеся системой доставки сообщений
- информация, использующееся подсистемами

Большенство систем доставки сообщений и библиотек поддерживают раздельное хранение метаданных и основной информации.

Формат информационной части сообщения никак не ограничивается фреймворками. Но информацию можно разделить на три группы:

- команды (Command Message)
- события (Event Message)
- данные (Document Message)

Любое сообщение имеет формат. Форматы можно условно разделить на следующие классы:

- транспортный (адаптеры канала, конечная точка)
- синтаксис (шаблонизаторы, xml заторы)
- тип данных (имена полей и т.д. xslt)
- семантика данных (кастомный код)


