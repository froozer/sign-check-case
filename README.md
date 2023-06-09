# sign-check-case

## Ситуация AS-IS

Пользователи сервиса N иногда оставляют ЭЦП (усиленную, квалифицированную). Для проверки подлинности этой ЭЦП используется сервис check-sign, который часть проверки осуществляет с помощью сервиса nalog.ru. Схематично взаимодействие представлено ниже:

![as-is](https://user-images.githubusercontent.com/17998108/225346663-ca91dd91-9e3b-46f4-8147-a3152d2669fe.PNG)

* microservice N взаимодействует с check-sign через сообщения в kafka. Топик "check-sign" - запрос на проверку, "success" - успешная проверка, "fail" - неуспешная
* сервис check-sign для отказоустойчивости развернут в 3 репликах. количество под может быть автоматически увеличено под нагрузкой
* взаимодействие check-sign и nalog.ru - через REST API, другие способы интеграции отсутствуют


## Проблема



![problem](https://user-images.githubusercontent.com/17998108/225348252-ee2e4446-55ed-4606-b172-36202ddf09b7.PNG)


Проблемой является периодическая недоступность сервиса nalog.ru. Требуется реализовать возможность накопить сообщения и, когда сервис восстановится, закончить проверку.


### Решение 1

![image](https://user-images.githubusercontent.com/17998108/225352623-9e9a1799-40ea-4515-8ffa-cca087230abd.png)

Необработанные подписи складываются в БД. Внешний шедулер периодически вызывает запрос на "переотправку" проблемных подписей.

**Минусы**
 - Очередь может разгребать только 1 инстанс check-sign
 - Если вызывать слишком часто - 2 инстанса начнут обрабатывать одну и ту же подпись
 
 
### Решение 2

![image](https://user-images.githubusercontent.com/17998108/225354515-bbc861a8-0bf3-44cc-9cf4-13bbb6eec0bd.png)

Необработанные подписи отправляются в топик DLQ. Из этого топика чтение происходит с задержкой и пачками по N записей.

**Минусы**
 - Еще один топик
 - Если сервис восстановится - разбор очереди будет медленный
 
 
### Решение 3

![image](https://user-images.githubusercontent.com/17998108/225356185-99d58260-8c96-414c-a164-07cc9e6f4d06.png)

Необработанные подписи возвращаются в тот же топик. Чтобы не попасть в бесконечный цикл, на сервисе check-sign необходимо реализовать [предохранитель](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)

**Минусы**
 - Придется кодить предохранитель
