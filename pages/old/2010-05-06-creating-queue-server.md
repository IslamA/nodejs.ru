# Создание сервера очередей

Для моего следующего проекта мне потребовался сервер очередей для Node.js. В принципе, его можно было применить уже в спайдере, но я не стал в тот раз заморачиваться. Теперь я решил наверстать упущенное и сделать настоящий отдельный сервер очередей.

В чём вообще суть сервера очередей? Он должен управлять задачами на выполнение. Сервер хранит список задач, раздаёт их рабочим процессам, и учитывает выполнение. В принципе, ничего сложного. Я решил сделать его отдельным TCP-сервером, чтобы неполадки рабочих процессов не сказывались на состоянии очереди (в пауке ошибка скрипта приводит к безвозвратной потере очереди URL на обработку).

Сервер будет состоять из двух компонент — собственно скрипт сервера, принимающий и отдащий задачи, и подключаемый модуль, оборачивающий работу с сервером в три удобные функции.

Вообще говоря, вначале я смотрел в сторону готовых серверов очередей, но в принципе не нашёл ничего подходящего (с полностью рабочим клиентом). В будущем возможно удобнее будет допилить модуль для общения с beanstalkd, но пока в качестве самообразования я хочу написать такой сервер самостоятельно.

## Пишем сервер

Простейший сервер состоит из TCP-сервера и массива с заданиями. По определённому сообщению от клиента задание помещается в очередь, по другому сообщению — забирается из очереди и отдаётся клиенту. Здесь нас ждёт два сюрприза. Во первых, если клиент отправляет два TCP-сообщения подряд, они могут прийти серверу как одно (их надо как то разделять). Во вторых, надо как то обрабатывать ошибки клиентов — например, рабочий процесс взял задание и умер. При этом задание надо вернуть в очередь.

С первой проблемой справиться довольно легко. Я даже не стал заморачиваться двоичным протоколом обмена — сервер и клиент отправляют друг другу обычный JSON. Простейшая функция смотрит, не содержится ли в сообщении несколько JSON-записей, и разрезает их если это необходимо. Вторая проблема посложнее. Я решил её так: когда клиент берёт сообщение, оно заносится в специальный массив "текущих заданий" вместе с идентификатором соединения с клиентом. Если соединение с клиентом во время выполнения потеряно, значит он умер, и мы возвращаем задачу в очередь. Очевидный минус такого подхода — клиент должен держать соединение с сервером пока задача не будет выполнена (а потом послать сигнал об успешном завершении задания).

Сервер выдаёт задания в порядке очереди — FIFO. Но т.к. задания приходят в виде JSON, к ним можно цеплять любые метаданные — например, приоритет. С этим я тоже не стал заморачиваться, мне это пока не нужно.

## Пишем клиентский модуль

Модуль для подключения клиента тоже несложный: он предоставляет три функции, add_job, get_job и notify для уведомления сервера о выполнении задания. Из них принимает callback только get_job — остальные функции даже не ожидают от сервера подтвержения (это, впрочем, легко исправить, если понадобится). Также такая схема должна неплохо уживаться с [псевдо-потоками](http://kuroikaze85.wordpress.com/2010/04/15/node-js-pseudo-threads/), если у каждого потока будет своё соединение с сервером очередей.

Работа с модулем выглядит примерно вот так:

```javascript
var queue = require('./work-queue'),
    sys = require('sys');
 
queue.connect(8999, '192.168.175.128', function(err, queue_conn) {
 
    sys.puts('Connected to queue');
 
    sys.puts('Sending a job to queue');
    // Отправляем в очередь новое задание
    queue_conn.add_job('Make clear glass goblet');
 
    setTimeout(function () {
        // Через некоторое время получаем задание.
        // Задержка здесь нужна только для наглядности, работать будет и так
        queue_conn.get_job(function(err, message) {
            if (!err) {
                sys.puts('Got job: ' + message);
            } else {
                sys.puts('No work to do');
            }
        });
 
    }, 2000);
 
});
```

Синтаксис функции get_job следует принятым в node.js соглашениям — последним параметром передаётся callback, который принимает первым параметром флаг ошибки. Пока что флаг ошибки установленный в true означает только одно — нет работы для выполнения. Если мы запросим задание из пустой очереди, мы получим именно этот результат.

Вообще говоря, веб-паук должен был изначально использовать именно такую схему организации очереди. После небольшой правки кода сервера можно хранить очередь не только в памяти, но и дублировать её в хранилище типа Redis или MongoDB, чтобы даже при перезапуске сервера выполнение заданий продолжилось с того же места, на котором закончилось. Примерный процесс работы выглядел бы так: паук получает URL от сервера очередей, обрабатывает его и отдаёт исходящие URL'ы обратно в качестве заданий. Проверка уникальности заданий в этом случае остаётся на стороне сервера.

Такая схема более-менее устойчива к ошибкам и внезапным перезагрузкам, и должна обеспечивать хорошее горизонтальнео масштабирование: для клиента совершенно прозрачно, на какой машине работает сервер очередей. Для моих целей такого сервера вполне достаточно, но для более серьёзной работы стоит посмотреть в сторону того же [beanstalk](http://kr.github.com/beanstalkd/). Текущую реализацию можно доработать, добавив подтверждения - об успешном добавлении нового задания, об успешном вычёркивании выполненого и т.д., и превращения соответствующих функций модуля в традиционные асинхронные.

Источник: [Механический мир](http://kuroikaze85.wordpress.com/2010/05/05/nodejs-work-queue-server/)