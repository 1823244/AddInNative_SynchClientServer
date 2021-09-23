# SynchClientServer library

![Platform](https://img.shields.io/badge/Platform-win--32%7C64-lightgrey)
[![License](https://img.shields.io/pypi/l/OPCDataTransfer)](https://en.wikipedia.org/wiki/MIT_License)
[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/Shanginre/AddInNative_SynchClientServer)](https://github.com/Shanginre/AddInNative_SynchClientServer/releases)

Внешняя Native API компонента для 1C 8.3, которая реализует синхронный TCP сервер. Основное назначение компоненты - 
работа в режиме TCP сервера, но она также может использоваться как TCP клиент. Компонента реализована на C++ 
с использованием библиотек [Boost](https://www.boost.org/). Работа с сокетами реализована с использованием библиотеки 
[Boost.Asio](https://www.boost.org/doc/libs/1_75_0/doc/html/boost_asio/overview.html).

Методы сервера реализованы в синхронном, а не в асинхронном варианте, чтобы упростить отладку и доработку компоненты. 
Библиотека Boost.Asio выбрана поскольку она сочетает в себе максимальную гибкость работы с сокетами, 
производительность и надежность. Так же в Boost.Asio заложены возможности использования не только TCP сокетов, но также
UDP-сокетов и COM-портов, что позволяет с минимальными изменениями в структуре кода адаптировать компоненту 
под специфику своей задачи.

Сервер TCP может работать с произвольным количество соединений как на одном, так и на разных портах (что не позволяет делать 
стандартная MSWINSCK.OCX от Microsoft). Компонента так же реализует много фич, которых нет в MSWINSCK.OCX. Например, в зависимости 
от сценария для каждого порта можно настраивать различные параметры:
- задержка чтения данных из буфера сокета - применяется, когда в сообщении нет завершающего символа. В данном случае, чтобы 
считать сообщение целиком, нужно сделать временную задержку от момента появления данных в буфере сокета, до момента полной 
загрузки данных в буфер. Как правило, такая задержка составляет не более 100 мс.
- задержка записи данных в сокет - применяется, когда нужно отправить пакет сообщений, а принимающая сторона способна 
их корректно обрабатывать, если сообщения отделены друг от друга некоторым временным лагом.
- количество секунд жизни сокета без активности сокета.

Компонента протестирована в ходе множества реальных внедрений и стабильно работает с большим количеством подключений и 
интенсивным трафиком сообщений. Основным сценарием использования компоненты в продакшене была интеграция медицинского
оборудования и лабораторной информационной системой на платформе 1С.

## Поддерживаемые системы
- Windows 32-бит
- Windows 64-бит

## Список методов
#### SetServerParameters(ПараметрыКомпонентыСервераJson)
Задает параметры работы сервера, но не запускает его. Если параметры сервера указаны не корректно, то выдается ошибка.

Параметры:
- ПараметрыКомпонентыСервераJson
    - ip (строка) - IP адрес на котором будет работать TCP сервер
    - loggingRequired (Булево) - признак логирования компоненты.
    - logFileName (строка) - полное имя файла логов компоненты. Имеет смысл только в режиме логирования компоненты.
    - memoryCleaningFrequency (Число) - частота (мс) очистки не актуальных и обработанных данных внутри компоненты. 
    Рекомендованное значение 5000 (5 сек).
    - ports (массив)
        - portType (строка) - строковое имя типа порта. Поддерживается список значений: TCP, UDP, COM. Для типов UDP, COM
        функционал пока не реализован.
        - portNumber (Число) - номер порта.
        - isLinkToRemoteServer (Булево) - признак того, что сокет на этом порте открыт другим удаленным сервером. В данном
        случае при общее через этот порт компонента будет выступать в роли клиента.
        - remoteIP (строка) - IP адрес удаленного сервера. Имеет смысл только если для порта указан признак isLinkToRemoteServer.
        - delayReadingFromSocket (строка) - задержка чтения данных из буфера сокета - применяется, когда в сообщении нет завершающего символа.
        - delayMessageSending (строка) - задержка записи сообщения в сокет при отправке нескольких сообщений в один сокет.
        - allowedTimeNoActivity (строка) - количество секунд жизни сокета без активности сокета.

#### Listen()
Запускает сервер TCP. На каждом порте сервера, где isLinkToRemoteServer = ЛОЖЬ, открываются сокеты, которые ждут подключений клиентов.
Если на каком-то порте сокет уже открыт другим сервером, тогда сервер не стартует, а возвращается строка JSON с описанием ошибки.

#### GetMessagesFromClientsWhenArrive(ВходящиеСообщенияJson, ЧастотаПроверки_мс, Таймаут_мс)
Записывает в переменную ВходящиеСообщенияJson строку JSON с данными о принятых сообщениях, которые не были еще обработаны. 
Функция возвращается ИСТИНА, если есть входящие сообщения или по истечение времени Таймаут_мс. Функция предназначена для
использования в качестве условия в цикле, например:

``` bsl
ВходящиеСообщенияJson = "";	
Пока КомпонентаСервер.GetMessagesFromClientsWhenArrive(ВходящиеСообщенияJson, ЧастотаПроверки_мс, Таймаут_мс) Цикл
    ...
КонецЦикла;
```

Параметры:
- ВходящиеСообщенияJson (строка) - переменная, в которую будет записана строка JSON с данными о принятых сообщениях. 
Если сообщений входящих нет, то будет пустая строка.
- ЧастотаПроверки_мс (Число) - частота (мс) проверки новых входящих сообщений
- Таймаут_мс (Число) - количество времени (мс) по истечении которого, если не было получено сообщений, то функция возвращает
ИСТИНА и поток управления передается на сторону 1С.

#### GetMessagesFromClients(ВходящиеСообщенияJson)
Записывает в переменную ВходящиеСообщенияJson строку JSON с данными о принятых сообщениях, которые не были еще обработаны. 
Функция возвращается ИСТИНА, если есть входящие сообщения. Функция предназначена для периодического вызова, например 
со стороны клиента 1С через обработчик ожидания. Нужно отметить, что функция GetMessagesFromClients выполняется 
значительно быстрее, чем функция GetMessagesFromClientsWhenArrive, но ее использование как условия в цикле "Пока" 
крайне нежелательно, так как это приведет к возрастанию загрузки процессора.

Параметры:
- ВходящиеСообщенияJson (строка) - переменная, в которую будет записана строка JSON с данными о принятых сообщениях. 
Если сообщений входящих нет, то будет пустая строка.

#### AckMessagesReceipt(МассивНомеровПринятыхСообщенийJson)
Передает компоненте, GUID принятых сообщений, чтобы они были помечены как обработанные. Обработанные сообщения не смогут 
быть вновь получены методами GetMessagesFromClientsWhenArrive и GetMessagesFromClients. В дальнейшем обработанные сообщения 
будут асинхронно удалены из памяти компоненты заданием, выполняемым с периодичностью memoryCleaningFrequency.

Параметры:
- МассивНомеровПринятыхСообщенийJson (строка) - строка JSON массивом GUID принятых сообщений.
    - ackMessagesUuid (массив)
        - messageUuidString (строка) - GUID принятого сообщения.

#### SendMessageToClient(СообщениеJson)
Передает компоненте сообщение для последующей отправки клиенту, подключенному к сокету сервера. Исходящие сообщения 
в дальнейшем будут обработаны и отправлены асинхронно в отдельном потоке.

Параметры:
- СообщениеJson (строка) - строка JSON с данными исходящего сообщения.
    - messageUuidString (строка)  - GUID исходящего сообщения.
    - portNumber (число) - номер порта получателя (равен номеру порта входящего сообщения).
    - clientSocketUuidString (строка) - GUID сокета получателя (равен GUID сокета входящего сообщения).
    - messageBody (строка) - тело сообщения. Управляющие символы ASCII будут передаваться в формате UNICODE. 
    Например, символ ACK (6 HEX) будет представлен как \u0006. 
     
     
#### SendMessageToRemoteServer(СообщениеJson)
Передает компоненте сообщение для последующей отправки в сокет, открытый удаленным сервером. Исходящие сообщения 
в дальнейшем будут обработаны и отправлены асинхронно в отдельном потоке.       
        
Параметры:
- СообщениеJson (строка) - строка JSON с данными исходящего сообщения.
    - messageUuidString (строка)  - GUID исходящего сообщения.
    - portNumber (число) - номер порта удаленного сервера.
    - messageBody (строка) - тело сообщения. Управляющие символы ASCII будут передаваться в формате UNICODE.
        
    
#### GetClientsState()
Возращает информацию о текущем состоянии сокетов на портах.

Возвращаемое значение (строка JSON):
- clientsConnections (массив)
    - clientSocketUuidString (строка) - GUID сокета.
    - lastActivityTime (строка) - последнее время чтения/записи данных в формате ISO.
    - portNumber (число) - номер порта.
    - accepted (булево) - признак подключения клиента к сокету. ИСТИНА - клиент подключен к сокету, ЛОЖЬ - ни один клиент
    пока не был подключен к сокету.

#### GetLastLogRecords(КоличествоЗаписей, ТолькоНовые)
Получает записи логов сервера. Полученные записи можно сохранить в регистре сведений для удобства просмотра.

Параметры:
- КоличествоЗаписей (число) - количество последних записей логов, которые требуется получить. Если параметр не задан, тогда
возвращаются все записи.
- ТолькоНовые (булево) - признак отбора записей логов. ИСТИНА - возвращаются только новые записи, ЛОЖЬ - возвращаются все записи.

Возвращаемое значение (строка JSON):
- logHistory (массив)
    - type (строка) - тип записи лога. Доступны значения: INFO, ERROR.
    - logRecordUuidString (строка) - GUID записи лога.
    - time (строка) - дата создания записи лога в формате ISO.
    - body (строка) - текст записи лога. Управляющие символы ASCII будут передаваться в формате UNICODE. 

#### StopServer()
Останавливает сервер и закрывает все сокеты.

## Примеры

В файле Работа_с_компонентой_SynchClientServer_Демо.cf в папке bin находится демонстрационная конфигурация 1C 8.3, содержащая 
примеры методов работы с компонентой.

#### Работа компоненты на стороне сервера 1С в фоновом задании.

``` bsl
Процедура ЗапуститьСервер() Экспорт
    
    ПараметрыСервера = ИнициализироватьСервер();	
    Если ПараметрыСервера = Неопределено Тогда
        ТекстОшибки = "Сервер не может быть запущен. Параметры сервера не определены";
        бит_КомпонентаSynchServerСервер.ДобавитьЗаписьЛога(ТекстОшибки, "ERROR");
        
        Возврат;
    КонецЕсли;

    ЗапуститьЦиклОбработкиСообщенийСервера(ПараметрыСервера.КомпонентаСервер, ПараметрыСервера.ПараметрыРаботыСервера);		
    
КонецПроцедуры;
        
Процедура ЗапуститьЦиклОбработкиСообщенийСервера(КомпонентаСервер, ПараметрыРаботыСервера)
    
    ЧастотаОбновленияСлужебнойИнформацииВБазе_сек = 10;
    ПоследнееОбновлениеСлужебнойИнформацииВБазе = ТекущаяДата();	
    
    ВходящиеСообщенияJson = "";		
    ЧастотаПроверки_мс = 1000;
    Таймаут_мс = 10000;
    Пока КомпонентаСервер.GetMessagesFromClientsWhenArrive(ВходящиеСообщенияJson, ЧастотаПроверки_мс, Таймаут_мс) Цикл
        Если ЗначениеЗаполнено(ВходящиеСообщенияJson) Тогда
            ВходящиеСообщенияСтруктура = ПрочитатьСтрокуJsonВСтруктуру(ВходящиеСообщенияJson, Истина);
            ВходящиеСообщенияJson = "";
            
            Если ЗначениеЗаполнено(ВходящиеСообщенияСтруктура) Тогда 			
                ПодтвердитьПолучениеСообщений(КомпонентаСервер, ВходящиеСообщенияСтруктура);
                
                ОбработатьВходящиеСообщения(
                    КомпонентаСервер, 
                    ВходящиеСообщенияСтруктура, 
                    ПараметрыРаботыСервера
                );
            КонецЕсли;
                                    
        КонецЕсли;
        
        Если ТекущаяДата() - ПоследнееОбновлениеСлужебнойИнформацииВБазе > ЧастотаОбновленияСлужебнойИнформацииВБазе_сек Тогда 
            ПоследнееОбновлениеСлужебнойИнформацииВБазе = ТекущаяДата();
            бит_КомпонентаSynchServerСервер.ОбновитьИнформациюСостоянияСокетовВБазеДанных(КомпонентаСервер);		
            бит_КомпонентаSynchServerСервер.СохранитьЛогиСервераВБазу(КомпонентаСервер);
        КонецЕсли;
    КонецЦикла;
    
КонецПроцедуры
```

#### Работа компоненты на стороне клиента 1С.

``` bsl
&НаКлиенте
Процедура ЗапуститьСерверНаКлиенте(Команда)

    ПараметрыСервера = бит_КомпонентаSynchServerКлиентСервер.ИнициализироватьСервер();	
        Если ПараметрыСервера = Неопределено Тогда
            ТекстОшибки = "Сервер не может быть запущен. Параметры сервера не определены";
            бит_КомпонентаSynchServerСервер.ДобавитьЗаписьЛога(ТекстОшибки, "ERROR");
            
            Возврат;
        КонецЕсли;
    
    ПодключитьОбработчикОжидания("ОбработатьНовыеСообщения", 1);

КонецПроцедуры
    
&НаКлиенте
Процедура ОбработатьНовыеСообщения() Экспорт

    бит_КомпонентаSynchServerКлиентСервер.ОбработатьНовыеСообщения(
        ПараметрыСервера.КомпонентаСервер, ПараметрыСервера.ПараметрыРаботыСервера
    );

КонецПроцедуры
    
&НаКлиенте    
Функция ОбработатьНовыеСообщения(КомпонентаСервер, ПараметрыРаботыСервера) Экспорт
    
    ВходящиеСообщенияJson = "";
    КомпонентаСервер.GetMessagesFromClients(ВходящиеСообщенияJson);
    Если ЗначениеЗаполнено(ВходящиеСообщенияJson) Тогда
        ВходящиеСообщенияСтруктура = ПрочитатьСтрокуJsonВСтруктуру(ВходящиеСообщенияJson, Истина);
        
        Если ЗначениеЗаполнено(ВходящиеСообщенияСтруктура) Тогда 			
            ПодтвердитьПолучениеСообщений(КомпонентаСервер, ВходящиеСообщенияСтруктура);
            
            ОбработатьВходящиеСообщения(
                КомпонентаСервер, 
                ВходящиеСообщенияСтруктура, 
                ПараметрыРаботыСервера
            );
        КонецЕсли;
    КонецЕсли;
    
    ВходящиеСообщенияJson = "";
    
КонецФункции
    
```

## Установка в конфигурацию
1. Скачать архив с релизом компоненты.
2. Загрузить в конфигурацию в качестве общего макета с двоичными данными

## Разворачивание окружения разработки на Windows
1. Установить Visual Studio 2017 и выше. При установке выбрать среди обязательных компонентов также Windows SDK 10. 
2. Скачать исходники репозитория данного проекта. 
3. Запустить файл проекта AddInNative_SynchClientServer.sln в каталоге AddInNative_SynchClientServerWindows.
4. Скачать набор библиотек [Boost](https://www.boost.org/users/download/). Проверенная версия 1.71.0. 
5. Собрать файлы .lib библиотек Boost с помощью утилиты b2.exe для платформ win32 и x64. Файлы для разных платформ 
помещаются в разные папки. Указать пути к каталогам библиотек в свойствах проекта.
6. Скачать библиотеку [rapidjson](https://github.com/Tencent/rapidjson/) для работы с JSON. Указать пути к каталогам 
библиотеки в свойствах проекта.
7. Указать пути к папке include, где содержатся файлы заголовков Native API.

## Планы доработки компоненты
1. Создать версию компоненты для операционных систем семейства linux.
2. Реализовать функционал работы с портам UDP и COM.
3. Реализовать возможность использования символов окончания сообщения, для того чтобы прекращать чтение данных из буфера 
сокета после того как такой символ был прочитан (сейчас применяется параметр delayReadingFromSocket).
    
Фичи ждут своих контрибьютеров.

## Как принять участие в разработке

Если Вы нашли какую-нибудь ошибку или хотите предложить доработку проекта, то порядок следующий:
1. Возьмите в работу какой-то issue или заведите новый issue.
2. Сделайте fork репозитория через кнопку в интерфейсе репозитория github-а.
3. Склонируйте репозиторий себе на машину.
4. На основании ветки master создайте новую ветку с номером задачи, например:
```
git checkout -b issue-9999
```
5. Выполните необходимые доработки и зафиксируйте необходимые изменения в исходниках в git.
6. Зайдите в репозиторий на github в векту мастер и создайте pull реквест через кнопку New pull request на вкладке pull requests.
7. После проверки pull реквеста, он либо будет отправлен Вам на доработку, или принят в main ветку. 

## Используемые сторонние продукты

1. [Boost](https://www.boost.org/), в частности [Boost.Asio](https://www.boost.org/doc/libs/1_75_0/doc/html/boost_asio/overview.html) -
работа с TCP сокетами.
2. [Rapidjson](https://github.com/Tencent/rapidjson/) - работа с форматом JSON.