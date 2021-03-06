# 1C-S3connector

Требования: Версия БСП 3.1.6.118 и выше (на других не тестировалось)

## Основная цель разработки

Модуль разрабатывался в первую очередь для интеграции с подсистемой БСП "Работа с файлами" и хранилищем S3 Minio, но его можно также использовать полностью самостоятельно и для любого другого хранилища S3. Для этого сформируйте и заполните параметры функции MinioS3Connector.ПараметрыMinio() и дальше используйте, как хотите. Примеры использования можно посмотреть в расширении "S3Connector" в общем модуле "РаботаСФайламиВТомахСлужебный".

## Почему был выбран именно такой вариант разработки

Работа с файлами в подсистеме БСП "Работа с файлами" разбита на несколько слоёв, поэтому основная идея была привязаться к самому нижнему слою, чтобы минимизировать количество вносимых изменений внося изменения только в тех местах, где происходит конечная обработка логики. С одной стороны это увеличивает порог вхождения на доработку алгоритма, так как требуется погрузиться в типовой алгоритм работы подсистемы "Работа с файлами", с другой стороны, позволяет очень легко интегрировать решение в свои наработки не особо разбираясь, а также легко обновляться на новые версии БСП.

## Детали реализации идеи

После анализа стало понятно, что лучше всего привязаться к типовому алгоритму сохранения файлов в локальном хранилище. По факту я пытаюсь на нижнем слое работать только с теми параметрами, которые дает мне вендор и в зависимости от настроек «Тома» формирую дополнительную логику. Для этого в «Том» был добавлен параметр sd_МестоХраненияФайлов. Параметр определяет в каком хранилище храниться файлы (локально или в Minio). Если в Minio, то идет «своя» логика обработки данных.

## Что было реализовано

На момент написание этого комментария, удалось реализовать весь основной функционал (добавление файла через кнопку добавить и скрепку, перетаскивание файла, сохранение новой версии файла, удаление файла) переопределив только 4 процедуры из общего модуля РаботаСФайламиВТомахСлужебный": ЗаписатьДанныеФайлаВТом, ДанныеФайла, УдалитьФайл, ПолноеИмяФайлаВТоме.
Также доработан штатный перенос файлов из локального тома и из информационной базы в хранилище S3. Для этого в расширении переопределены пару процедур в обработки ПереносФайлов.

## Нюансы работы алгоритма

Указанная версия БСП при хранении файлов в локальных томах умеет:
 1. Сохранять файлы в разбивке по дням, т. е. если я возьму один и тот же файл и сохраню его сегодня и завтра, то храниться он будет в разных каталогах с именем "дата".
 2. Сохранять один и тот же файл за один и тот же день в разных каталогах. Обеспечивается это за счет функции Общий модуль РаботаСФайламиСлужебныйКлиентСервер.Функция ПолучитьУникальноеИмяСПутем(ИмяКаталога, ИмяФайла) в функции есть абсолютно дурацкая проверка на существование каталога. По факту алгоритм выбирает одну из 26 букАв и перебором проверяет есть ли такой каталог если каталог есть, то цикл повторяется.
 3. В зависимости от флажка "Создавать подкаталоги с именами справочников-владельцев файлов" в "Администрирование" - > "Настройка работы с файлами" к пути может еще прибавляться имя справочника владельца файла.

По второму пункту я решил не заморачиваться и вместо буКавки просто прибавляю время к текущему каталогу (процедура MinioS3Connector.СохранитьФайлВMinio(ПараметрыMinio)).

## Краткая инструкция по подключению модуля "Работа с файлами" БСП к новому справочнику

1. Создаем справочник в основной конфигурации (текущая версия БСП с установленным режимом совместимости не позволила сделать нормальное решение полностью на расширении)
2. Есть 2 возможности хранения файлов для справочника 1С:
   - в типовом справочнике БСП "Файлы" – я пока разобрался только с этим вариантом
   - в специально созданном справочнике с названием <Имя справочника>ПрисоединенныеФайлы – с этим вариантом пока не разбирался, поэтому возможно что моя доработка  с ним не работает.
3. Если мы хотим подключить типовой справочник БСП "Файлы" нужно в определяемые типы "ВладелецФайлов" добавить новый справочник.
4. Для того чтобы файл записывался штатно, его нужно записать в регистр "НаличиеФайлов", а это можно сделать, добавив новый справочник в определяемые типы "ВладелецПрисоединенныхФайлов"
5. Для того чтобы при копировании элемента в вашем справочнике копировались и присоединенные файлы, нужно в форме элемента добавить параметр "ЗначениеКопирования", тип: ссылка на этот же справочник и нужно поставить флажок "Ключевой параметр".
6. В модуль формы справочника нужно добавить несколько процедур (см. Example.bsl).
7. Процедуры из раздела ОбработчикиСобытийФормы нужно привязать к соответствующим событиям формы


## Немного о копирайтах

1.	Как обычно для взаимодействия по HTTP был взят модуль Владимира Бондаревcкого https://github.com/vbondarevsky/Connector. К сожалению, взял пока не последнюю версию, так как последняя падает при преобразовании структуры. Пока некогда было разбираться, будет время обновлю
2.	Идея о внедрении в типовой алгоритм подсистемы «Работа с файлами» была взята у моего коллеги Антона Петроненко
3.	Также были использованы наработки по работе с MinIO от мой коллеги Кристины Ватагиной

Получилось с мира по нитке, нам подсистема работы с MinIO😊.

Если что-то не так или непонятно пишите, подправлю.

PS: Модуль находится на этапе тестирования, поэтому возможны баги и все что угодно, поэтому пока прошу воспринимать разработку, как MVP 1 :-).
