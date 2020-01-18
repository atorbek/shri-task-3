# [ТЗ](TASK.md)

# Ход действий

## Установка зависимостей

Установил зависимости через `npm i`. В консоли появились предупреждения:

```
npm WARN shri-ext@0.0.1 No repository field.
npm WARN shri-ext@0.0.1 No license field.
```

1-й WARN пофиксил добавлением в package.json `repository`

```json
"repository": {
		"type": "git",
		"url": "https://github.com/atorbek/shri-task-3"
	},
```

Поля type и url нужны для контрибьюторов. `npm docs` по `url` откроет страницу в браузере. 


2-й WARN убрал указанием типа лицензии, которая позволяет использовать библиотеку в коммерческих проектах

```json
"license": "BSD-3-Clause"
```

Еще было сообщение о уязвимости в bem-xjst, оставил как есть, `npm audit fix` может не найти версию без уязвимости, но даже если найдет - поведение работы может отличаться.


## Запуск приложения

Запустил плагин в режиме дебага (через `F5`) - из-за ошибок приложение не поднялось:

<pre> Argument of type '(params: InitializeParams) => { capabilities: { textDocumentSync: string; }; }' is not assignable to parameter of type 'RequestHandler<InitializeParams, InitializeResult, InitializeError>'.
  Type '{ capabilities: { textDocumentSync: string; }; }' is not assignable to type 'HandlerResult<InitializeResult, InitializeError>'.
    Type '{ capabilities: { textDocumentSync: string; }; }' is not assignable to type 'InitializeResult'.
      Types of property 'capabilities' are incompatible.
        Type '{ textDocumentSync: string; }' is not assignable to type 'ServerCapabilities'.
          Type '{ textDocumentSync: string; }' is not assignable to type '_ServerCapabilities'.
            Types of property 'textDocumentSync' are incompatible.
              Type 'string' is not assignable to type '0 | TextDocumentSyncOptions | 1 | 2 | undefined'. </pre>

Изучил stacktrace, в значении `textDocumentSync` был невалидный тип. Прочитал про событие `onInitialize`.
Событие отрабатывает при открытии файла, и, в качесте результата, возвращает настройки на клиент. `textDocumentSync`
говорит клиенту как отправлять содержимое файла на языковой сервер - только измененную часть или полностью файл. Указал
`documents.syncKind`, чтобы возвращался полностью файл.

<pre>Property 'loc' does not exist on type 'AstIdentifier'.</pre>

 В `AstProperty` отсутствует путь  `key.loc`, заменил на `loc`.

  ```ts
export interface AstProperty {
        type: 'Property';
        key: AstIdentifier;
        value: AstJsonEntity;
        loc: AstLocation;
    }
 ```
## Превью интерфейса
 
Сначала решил разобраться как работает клиент:

 - Посмотрел как работает превью по заданию;
 - Просмотрел на какие события отрабатывает логика по отображению превью.

 Работает след. образом:
 
 - На изменение в файле отрабатывает событие;
 - По событию получаем значение json;
 - Преобразовываем json к html;
 - Обновляем содержимое превью если превью открыто;
 - Или открываем превью и добавляем содержимое.

После понимания, сначала прошелся по ошибкам линтера `tslint`:

- добавил точку с запятой в 
```ts 
const template = bemhtml.compile();
```
```ts 
e.dispose();
```
- поправил код `if (panel) panel.reveal();` в методе `openPreview` на 

```ts
  if (panel) {
      panel.reveal();
  } else {
      const panel = initPreviewPanel(document);
      updateContent(document, context);
      context.subscriptions.push(panel);
  }
```
Т.е если превью открыто, то обновленное превью должно открываться в той же вкладке.

Дальше, по заданию проверил, что превью открывается в отдельной вкладке:
 
 - При нажатии на кнопку сверху от редактора;
 - При нажатии горячих клавиш `Ctrl+Shift+V`; 
 - И при выполнении команды `Example: Show preview`.

Таким же способом проверил, что обновленное превью открывается в той же вкладке. 

На шаге с линтером заметил, что содержимое контента не отображается. Подебажил клиентский код. Обнаружил, что регулярка замены `{{content}}` на html, неправильно написана, потому что `s` - cоответствует одиночному символу пустого пространства, включая пробел, табуляцию, прогон страницы, перевод строки, а `s+` - символ s повторенный 1 или более раз.

Как решение, добавить пробельный символ в начало и конец слова `content` - `{{ content }}` либо переписать регулярку. Выбрал 2-й вариант.

```ts
panel.webview.html = previewHtml 
    .replace(/{{\s*(\w+)\s*}}/g, (str, key) => {
        switch (key) {
            case 'content':
                return html;
            case 'mediaPath':
                return getMediaPath(context);
            default:
                return str;
        }
    });
```

Исправление не совсем помогло. html появился в dom webview, но стили не подгрузились. Проверил через инспектор браузера в vscode, команда `>Developer: Open Webview Developer Tools`. 

Почитал документацию https://code.visualstudio.com/updates/v1_38#_webviewaswebviewuri-and-webviewcspsource,https://code.visualstudio.com/api/extension-guides/webview#content-security-policy в ней есть строки

> All webviews (even very simple ones) should set a content security policy. This helps limit the potential impact of content injections and is generally a good measure for defense in depth. We've documented how to add a content security policy to VS Code webviews in the Webview extension guide.

Поэтому, через `meta` в `index.html` расшарил доступ к изображениям, скриптам и стилям:

```html
<meta http-equiv="Content-Security-Policy" content="default-src vscode-resource:; img-src vscode-resource: https:; script-src vscode-resource:; style-src vscode-resource:;"/>
```
Но стили не подгрузились.

Через инспектор заметил, что схема в base не `vscode-resource`, а `resource`. Вспомнил, что есть метод `getMediaPath`, который подставляет путь вместо `{{mediaPath}}`. Заменил схему обращения на `vscode-resource`.

Cтили подгрузились!

Сразу решил подключить `css` и `javascript` из первого задания.

## Линтер структуры блоков

Сначала просмотрел как должно работать по заданию. Заметил, что линтер не подсвечивает ошибочное место в файле. Чтобы разобраться, решил настроить дебаг `server.ts`.

В документации https://code.visualstudio.com/api/language-extensions/language-server-extension-guide написано `we need to attach a debugger to the running server`, для этого:

- В `launch.json` описал к какому процессу нужно приатачиться

```json
		{
			"type": "node",
			"request": "attach",
			"name": "Attach to Server",
			"port": 6009,
			"restart": true,
			"outFiles": ["${workspaceRoot}/server/out/**/*.js"]
		},
```

- Затем переподнял плагин в дебаг режиме и попытался подключиться к процессу сервера.

 При попытке attach к серверу получил сообщение `Cannot connect to runtime process, timeout after 10000 ms - (reason: Cannot connect to the target: connect ECONNREFUSED 127.0.0.1:6009)`. Скорее всего, процесс сервера не поднялся.

Пересмотрел еще раз код `extension.ts`, нашел место, где поднимается клиент и сервер. В коде есть строка `new SettingMonitor(client, 'example.enable').start()`, понял, что в настройках плагина нужно проставить `example.enable` в `true`, тогда запустится сервер.

После запуска сервера появилась ошибка о невалидном json:

```
(node:27684) UnhandledPromiseRejectionWarning: SyntaxError: Unexpected symbol <f> at 1:1
1 | file:///home/alexander/Documents/projects/GROUP%201/test.json
    ^
    at Object.<anonymous> (/home/alexander/Documents/projects/shri-task-3/node_modules/json-to-ast/build.js:5:2)
    at Module._compile (internal/modules/cjs/loader.js:786:30)
    at Object..js (internal/modules/cjs/loader.js:798:10)
    at Module.load (internal/modules/cjs/loader.js:645:32)
    at Function._load (internal/modules/cjs/loader.js:560:12)
    at Module.require (internal/modules/cjs/loader.js:685:19)
    at require (internal/modules/cjs/helpers.js:16:16)
    at Object.<anonymous> (/home/alexander/Documents/projects/shri-task-3/src/linter.ts:1:1)
```

В метод `parseJson(json)` на вход передавали не содержимое файла, а путь к нему. Заменил `textDocument.uri` на `textDocument.getText();`.

После исправления ошибки, линтер все еще не подсвечивал ошибки.

Продебажил код, убедился, что `validateProperty(property)` и `validateObject(obj)` возвращает массив ошибок, однако массив `errors` все еще был пустым. Посмотрел как добавляются элементы в `errors`, увидел `concat` вместо `push`, заменил на `push`:

```ts
walk(ast, 
    (property: jsonToAst.AstProperty) => errors.push(...validateProperty(property)), 
    (obj: jsonToAst.AstObject) => errors.push(...validateObject(obj)));
```

Подключил линтер из 2-го задания.

После исправления последней ошибки в `json`, линтер все еще ее подсвечивает. Исправил удалением условия `if (errors.length) { }` в методе `validateTextDocument`. Теперь метод `sendDiagnostics`, вызывается даже, когда массив `errors` пустой.

Заметил, что после закрытия json-файла c ошибками на вкладке `PROBLEMS` остаются ошибки закрытого файла. Сравнил с линтером `tslint`. В `tslint` после закрытия файла с ошибками - ошибки пропадают, поэтому добавил код, который очищает `PROBLEMS`
после закрытия json-файла с ошибками:

```ts
docs.onDidClose(change => {
    conn.sendDiagnostics({ uri: change.document.uri, diagnostics: [] });
});
```

Внес дополнительные правки:

- В событии `onInitialize` удалил неиспользуемый параметр `params: InitializeParams`;
- Добавил переменной `docs` тип `TextDocuments`;
- В методе `GetSeverity` в `case Severity.Error` поправил `DiagnosticSeverity.Information` на `DiagnosticSeverity.Error`;
- В методе `updateContent`, пробросил ошибку в `panel.webview.html`;
- Удалил неиспользуемый импорт `DocumentColorRequest`.
