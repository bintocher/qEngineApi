# qEngineApi

### Библиотека для подключения к engine api qlik sense
Ссылка проекта на [pypi]
### Установка
```sh
pip install qEngineApi
```
### Используемые библиотеки
- ssl
- os
- websocket
- websocket-client
- json
### Как работать с библиотекой

#### подключение библиотеки
```python
from qEngineApi import QlikEngine
```
#### создание подключения.
```python
engineConnect = QlikEngine(host = "HOST" ,cert_path =  "path:/to/dir", user_directory = "DIRECTORY", user_id = "USER")
```
в параметрах подключения необходимо указать:
- **host** -  адрес вашего сервера, где развернут qlik. Убедитесь что у машины, с которой вы будете отправлять запросы есть доступ к серверу qlik на порт **4747**.
- **cert_path** - абсолютный путь до папки с сертификатами. Как выгрузить сертификаты, можно почитать [тут]. Экспортировать необходимо в формате **Platform independent PEM-format**. После экспорта появится три файла: root.pem, client.pem, client_key.pem. Все они должны лежать по указанному пути, сохраняя оригинальные именования.
- **user_directory** - директория вашего пользователя (домен учетной записи)
- **user_id** - ваш пользователь

Можно использовать внутренню учетную запись QS: user_directory="internal", user_id="sa_repository"

**!!!Важно -на пользователя под которым вы подключаетесь распространяются все ограничения, как на доступ к документам/действиям, так и на доступ к данным внутри приложений в случае если включена section access. То есть доступ в engine api будет полностью соответствовать доступу в hub указанного пользователя. Указать можно любого пользователя под которым вы хотите работать.**

после создания подключения можно проверить статус подключения в параметре **engineConnect.sessionCreated**, Все последующие действия рекомендуется выполнять только если он **true**. После окончания работы с engine api необходимо закрыть соединение вызвав **del engineConnect**
```python
sessionCreated = engineConnect.sessionCreated
if sessionCreated:
    //ваш код
    ...
    del engineConnect #убиваем сессию после того как завершили все действия
```
### Описание методов
#### **getDocList(self)** - возвращает список документов доступных пользователю
```python
documents = engineConnect.getDocList()
print(json.dumps(documents))
```
#### **openDoc(self, docId)** - открывает подключение к конкретному приложению
```python
doc = engineConnect.openDoc('2ec2ed23-a62f-4c9d-b501-1b3a5fac192c')
docHandle = doc['result']['qReturn']['qHandle']
print(json.dumps(doc))
```
в качестве аргумента необходимо передать id приложения qlik sense. Найти его можно либо в ссылке, открыв его в хабе, либо посмотреть в qmc, либо воспользоваться методом getDocList().
**!!!Для дальнейшей работы с открытым документом, необходимо запомнить handle этого вызова**

#### **applyBookmark(self,handle,bookmarkId)** - применяет указанную закладку к документу.
```python
bookmark = engineConnect.applyBookmark(docHandle,'c37ff895-d611-44b8-95b3-d6ddaa35f9a9')
print(json.dumps(bookmark))
```
в качестве аргумента необходимо передать handle, который вернулся в методе открытия документа И id закладки.

#### **getObject(self,handle,objectId)** - обратиться к конкретному объекту в приложении.
```python
object = engineConnect.getObject(docHandle,'PegSyT')
objectHandle = object['result']['qReturn']['qHandle']
```
В качестве аргумента необходимо передать handle, который вернулся в методе открытия документа и id объекта.
**!!!Для дальнейшей работы с объектом необходимо запомнить handle, вернувшийся в ответе.**

#### **exportData(self,handle,exportFormat)** - метод для экспорта данных объекта, вернет json содержащий ссылку на скачивание файла.
```python
export = engineConnect.exportData(objectHandle,'OOXML')
downloadUrl = 'http://'+ host  + export['result']['qUrl']
print(downloadUrl)
```
В качестве аргументов необходимо указать handle, который вернулся в методе обращения к объекту и формат экспорта.
Подробнее о форматах можно почитать в [справке qlik]

#### **getSheetsObject(self,handle,exportFormat)** - вернет массив всех листов приложения со всеми их объектами.
```python
sheetsObject= engineConnect.getSheetsObject(docHandle)
sheets = sheetsObject['result']['qLayout']['qAppObjectList']['qItems']
for sheet in sheets:
     sheetObjects = sheet['qData']['cells']
      for object in sheetObjects:
         print(object)
```
В качестве аргумента необходимо передать handle, который вернулся в методе обращения к документу.

#### **showVariables(self, handle, showreserve = False, showConfig = False)** - вернет список переменных со всеми атрибутами и выведет в консоль список переменных с несколькими атрибутами (isScriptCreated, qIncludeInBookmark, id, name, definition)
```python
variables = engineConnect.showVariables(docHandle,False,False)
```
вывод в консоль:
```sh
isScriptCreated qIncludeInBookmark id                                       name                           definition
false           true               954d3849-1cf6-45e4-884f-65cc1b9fa193     vVariable3                     variable3
false           true               a64cb8ad-594d-442d-9292-897d4c9acd22     vVariable4                     variable4
true            false              3269f257-0cd0-4efa-bce6-854e74bb2931     vVariable1                     variable1
```

в качестве аргументов необходимо передать handle, который вернулся в методе обращения к документу, а так же два boolean параметра отвечающих за вывод зарезевированных и конфигурационных переменных.

#### **includeVariableInBookmark(self,handle,include=True,variable="ALL")** - метод для регулирования включения/исключения переменных в закладки.

  **!!! В закладки можно включать только те переменные, которые были созданы в интерфейсе. Переменные объявленные в скрипте в закладках сохраняться не будут. Это ограничения движка qlik**
```python
engineConnect.includeVariableInBookmark(docHandle,True) #Позволит включать все переменные в закладки
engineConnect.includeVariableInBookmark(docHandle,False) #Переменные не будут включаться в закладки
```
если необхолимо определить свойство только для одной переменной, то ее имя необходимо указать третьим аргументом
```python
engineConnect.includeVariableInBookmark(docHandle,True, 'vVariable3') #Значение переменной vVariable3 будет сохранятся в закладках
engineConnect.includeVariableInBookmark(docHandle,False,'vVariable3') #Значение переменной vVariable3 не будет сохранятся в закладках
```
в качестве первого аргумента необходимо указать handle, который вернулся в методе обращения к документу

#### **sendRequest(self,request)** - произвольный запрос к engine api. Вернет json с ответом.
```python
request = {
    "handle": docHandle,
	"method": "ApplyBookmark",
	"params": {
	        "qId": "c37ff895-d611-44b8-95b3-d6ddaa35f9a9"
	}
}

result = engineConnect.sendRequest(request)
print(json.dumps(result))
```

В качестве аргумента можно передать любой произвольный запрос.
**Поэкспериментировать с запросами к engine api можно в [dev-hub->engine api explorer]. Попасть в него можно через hub->троеточие в верхнем правом углу страницы-> Dev-hub**

[//]: #
[тут]: <https://help.qlik.com/en-US/sense-admin/November2022/Subsystems/DeployAdministerQSE/Content/Sense_DeployAdminister/QSEoW/Administer_QSEoW/Managing_QSEoW/export-certificates.htm>
[pypi]: <https://pypi.org/project/qEngineApi/>
[справке qlik]: <https://help.qlik.com/en-US/sense-developer/November2022/Subsystems/EngineJSONAPI/Content/service-genericobject-exportdata.htm>
[dev-hub->engine api explorer]: <https://help.qlik.com/en-US/sense-developer/November2022/Subsystems/Dev-Hub/Content/Sense_Dev-Hub/EngineApiExplorer/engine-api-explorer.htm>
