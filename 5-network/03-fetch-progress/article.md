
# Fetch: ход загрузки

Метод `fetch` позволяет отслеживать процесс *получения* данных.

Заметим, на данный момент в `fetch` нет способа отслеживать процесс *отправки*. Для этого используйте [XMLHttpRequest](info:xmlhttprequest), позже мы его рассмотрим.

Чтобы отслеживать ход загрузки данных с сервера, можно использовать свойство `response.body`. Это `ReadableStream` ("поток для чтения") -- особый объект, который предоставляет тело ответа по частям, по мере поступления. Потоки для чтения описаны в спецификации [Streams API](https://streams.spec.whatwg.org/#rs-class).

В отличие от `response.text()`, `response.json()` и других методов, `response.body` даёт полный контроль над процессом чтения, и мы можем подсчитать, сколько данных получено на каждый момент.

Вот примерный код, который читает ответ из `response.body`:

```js
// вместо response.json() и других методов
const reader = response.body.getReader();

// бесконечный цикл, пока идёт загрузка
while(true) {
  // done становится true в последнем фрагменте
  // value - Uint8Array из байтов каждого фрагмента
  const {done, value} = await reader.read();

  if (done) {
    break;
  }

  console.log(`Получено ${value.length} байт`)
}
```

Результат вызова `await reader.read()` - это объект с двумя свойствами:
- **`done`** -- `true`, когда чтение закончено, иначе `false`.
- **`value`** -- типизированный массив данных ответа `Uint8Array`.

```smart
Streams API также описывает асинхронный перебор по `ReadableStream`, при помощи цикла `for await..of`, но он пока слабо поддерживается (см. [задачи для браузеров](https://github.com/whatwg/streams/issues/778#issuecomment-461341033)), поэтому используем цикл `while`.
```

Мы получаем новые фрагмента данных в цикле, пока загрузка не завершится, то есть пока `done` не станет `true`.

Чтобы отслеживать процесс загрузки, нам нужно при получении очередного фрагмента прибавлять его длину `value` к счетчику.

Вот полный рабочий пример, который получает ответ сервера и в процессе получения выводит в консоли длину полученных данных:

```js run async
// Шаг 1: начинаем загрузку fetch, получаем поток для чтения
let response = await fetch('https://api.github.com/repos/javascript-tutorial/en.javascript.info/commits?per_page=100');

const reader = response.body.getReader();

// Шаг 2: получаем длину содержимого ответа
const contentLength = +response.headers.get('Content-Length');

// Шаг 3: считываем данные:
let receivedLength = 0; // количество байт, полученных на данный момент
let chunks = []; // массив полученных двоичных фрагментов (составляющих тело ответа)
while(true) {
  const {done, value} = await reader.read();

  if (done) {
    break;
  }

  chunks.push(value);
  receivedLength += value.length;

  console.log(`Получено ${receivedLength} из ${contentLength}`)
}

// Шаг 4: соединим фрагменты в общий типизированный массив Uint8Array
let chunksAll = new Uint8Array(receivedLength); // (4.1)
let position = 0;
for(let chunk of chunks) {
	chunksAll.set(chunk, position); // (4.2)
	position += chunk.length;
}

// Шаг 5: декодируем Uint8Array обратно в строку
let result = new TextDecoder("utf-8").decode(chunksAll);

// Готово!
let commits = JSON.parse(result);
alert(commits[0].author.login);
```

Разберёмся, что здесь произошло:

1. Мы обращаемся к `fetch` как обычно, но вместо вызова `response.json()` мы получаем доступ к потоку чтения `response.body.getReader()`.

    Обратите внимание, что мы не можем использовать одновременно оба эти метода для чтения одного и того же ответа: либо обычный метод `response.json()`, либо чтение потока `response.body`.
2. Ещё до чтения потока мы можем вычислить полную длину ответа из заголовка `Content-Length`.

    Он может быть нечитаемым при запросах на другой источник (подробнее в разделе <info:fetch-crossorigin>) и, в общем-то, серверу необязательно его устанавливать. Тем не менее, обычно длина указана.
3. Вызываем `await reader.read()` до окончания загрузки.

    Всё, что получили, мы складываем по "кусочкам" в массив `chunks`. Это важно, потому что после того, как ответ получен, мы уже не сможем "перечитать" его, используя `response.json()` или любой другой способ (попробуйте - будет ошибка).
4. В самом конце у нас типизированный массив -- `Uint8Array`. В нём находятся фрагменты данных. Нам нужно их склеить, чтобы получить строку. К сожалению, для этого нет специального метода, но можно сделать, например, так:
    1. Создаём `chunksAll = new Uint8Array(receivedLength)` -- массив того же типа заданной длины.
    2. Используем `.set(chunk, position)` для копирования каждого фрагмента друг за другом в него.
5. Наш результат теперь хранится в `chunksAll`. Это не строка, а байтовый массив.

    Чтобы получить именно строку, надо декодировать байты. Встроенный объект [TextDecoder](info:text-decoder) как раз этим и занимается. Потом мы можем, если необходимо, преобразовать строку в данные с помощью `JSON.parse`.

    Что если результат нам нужен в бинарном виде вместо строки? Это ещё проще. Замените шаги 4 и 5 на создание единого `Blob` из всех фрагментов:
    ```js
    let blob = new Blob(chunks);
    ```

В итоге у нас есть результат (строки или `Blob`, смотря что удобно) и отслеживание прогресса получения.

На всякий случай повторимся, что здесь мы рассмотрели, как отслеживать процесс получения данных с сервера, а не их отправки на сервер. Для отслеживания закачки у `fetch` пока нет способа.
