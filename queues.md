git bd36d6978d15610746e6587deb8a1e9659628898

---

# Очереди

- [Настройка](#configuration)
- [Использование очередей](#basic-usage)
- [Добавление функций-замыканий в очередь](#queueing-closures)
- [Обработчик очереди](#running-the-queue-listener)
- [Push-очереди](#push-queues)
- [Незавершенные задачи](#failed-jobs)

<a name="configuration"></a>
## Настройка

В Laravel компонент Queue предоставляет единое API для различных сервисов очередей. Очереди позволяют вам отложить выполнение времязатратной задачи, такой как отправка e-mail, на более позднее время, таким образом на порядок ускоряя загрузку (генерацию) страницы.

Настройки очередей хранятся в файле `app/config/queue.php`. В нём вы найдёте настройки для драйверов-связей, которые поставляются вместе с фреймворком: [Beanstalkd](http://kr.github.com/beanstalkd), [IronMQ](http://iron.io), [Amazon SQS](http://aws.amazon.com/sqs), а также синхронный драйвер (для локального использования).

Упомянутые выше драйвера имеют следующие зависимости:

- Beanstalkd: `pda/pheanstalk`
- Amazon SQS: `aws/aws-sdk-php`
- IronMQ: `iron-io/iron_mq`

<a name="basic-usage"></a>
## Использование очередей

#### Добавление новой задачи в очередь

	Queue::push('SendEmail', array('message' => $message));

Первый аргумент метода `Queue::push` - имя класса, который должен использоваться для обработки задачи. Второй аргумент - массив параметров, которые будут переданы обработчику.

#### Регистрация обработчика задачи

	class SendEmail {

		public function fire($job, $data)
		{
			//
		}

	}

Заметьте, что `fire` - единственный обязательный метод этого класса; он получает экземпляр объект `Job` и массив данных, переданных при добавлении задачи в очередь.

Если вы хотите использовать какой-то другой метод вместо `fire` - передайте его имя при добавлении задачи.

#### Задача с произвольным методом-обработчиком

	Queue::push('SendEmail@send', array('message' => $message));

#### Добавление задачи в определенную очередь

У приложения может быть несколько очередей. Чтобы поместить задачу в определенную очередь, укажите её имя третьим аргументом:

	Queue::push('SendEmail@send', array('message' => $message), 'emails');

#### Передача данных нескольким задачам сразу

Если вам надо передать одни и те же данные нескольким задачам в очереди, вы можете использовать метод `Queue::bulk`:

	Queue::bulk(array('SendEmail', 'NotifyUser'), $payload);

Иногда вам нужно, чтобы задача начала исполняться не сразу после занесения её в очередь, а спустя какое-то время. Например, выслать пользователю письмо спустя 15 минут после регистрации. Для этого существует метод `Queue::later`:

#### Отложенное выполнение задачи

	$date = Carbon::now()->addMinutes(15);

	Queue::later($date, 'SendEmail@send', array('message' => $message));

Здесь для задания временного периода используется библиотека для работы с временем и датой Carbon, но `$date` может быть и просто целым числом секунд.


Как только вы закончили обработку задачи она должна быть удалена из очереди - это можно сделать методом `delete` объекта `Job`.

#### Удаление выполненной задачи

	public function fire($job, $data)
	{
		// Обработка задачи

		$job->delete();
	}

Если вы хотите поместить задачу обратно в очередь - используйте метод `release`:

#### Помещение задачи обратно в очередь

	public function fire($job, $data)
	{
		// Обработка задачи

		$job->release();
	}

Вы также можете указать число секунд, после которого задача будет помещена обратно:

	$job->release(5);

Если во время обработки задания возникнет исключение (exception), задание будет помещено обратно в очередь. Вы можете получить число сделанных попыток запуска задания методом `attempts`:

#### Получение числа попыток запуска задания

	if ($job->attempts() > 3)
	{
		//
	}

Вы также можете получить идентификатор задачи

#### Получение идентификатора задачи

	$job->getJobId();

<a name="queueing-closures"></a>
## Добавление функций-замыканий в очередь

Вы можете помещать в очередь и функции-замыкания. Это очень удобно для простых, быстрых задач, выполняющихся в очереди.

#### Добавить функцию-замыкание в очередь

	Queue::push(function($job) use ($id)
	{
		Account::delete($id);

		$job->delete();
	});

При использовании [push-очередей](#push-queues) Iron.io, будьте особенно внимательны при добавлении замыканий. Конечная точка выполнения, получающая ваше сообщение из очереди, должна проверить входящую последовательность-ключ, чтобы удостовериться, что запрос действительно исходит от Iron.io. Например, ваша конечная push-точка может иметь адрес вида `https://yourapp.com/queue/receive?token=SecretToken` где значение `token` можно проверять перед собственно обработкой задачи.

<a name="running-the-queue-listener"></a>
## Обработчик очереди

Задачи, помещенные в очередь должен кто-то исполнять. Laravel включает в себя Artisan-задачу, которая "слушает" очередь и выполняет новые задачи по мере их поступления (задачи запускаются не параллельно, а последовательно). Вы можете запустить её командой `queue:listen`:

#### Запуск сервера выполнения задач

	php artisan queue:listen

Вы также можете указать, какое именно соединение должно прослушиваться:

	php artisan queue:listen connection

Заметьте, что когда это задание запущено оно будет продолжать работать, пока вы не остановите его вручную. Вы можете использовать монитор процессов, такой как [Supervisor](http://supervisord.org/), чтобы удостовериться, что задание продолжает работать.

Вы можете указать, задачи из каких очередей нужно будет исполнять в первую очередь. Для этого перечислите их через запятую в порядке уменьшения приоритета:

	php artisan queue:listen --queue=high,low

Задачи из `high` будут всегда выполняться раньше задач из `low`.

#### Указание числа секунд для работы сервера

Вы можете указать число секунд, в течении которых будут выполняться задачи - например, для того, чтобы поставить `queue:listen` в cron на запуск раз в минуту.

	php artisan queue:listen --timeout=60

#### Уменьшение частоты опроса очереди

Для уменьшения нагрузки на очередь, вы можете указать время, которое сервер выполнения задач должен бездействовать перед опросом очереди. 

	php artisan queue:listen --sleep=5

Если очередь пуста, она будет опрашиваться раз в 5 секунд. Если в очереди есть задачи, они исполняются в штатном режиме, без задержек.

#### Обработка только первой задачи в очереди

Для обработки только одной (первой) задачи можно использовать команду `queue:work`:

	php artisan queue:work

<a name="push-queues"></a>
## Push-очереди

Push-очереди дают вам доступ ко всем мощным возможностям, предоставляемым подсистемой очередей Laravel 4 без запуска серверов или фоновых программ. На текущий момент push-очереди поддерживает только драйвер [Iron.io](http://iron.io). Перед тем, как начать, создайте аккаунт и впишите его данные в `app/config/queue.php`.

После этого вы можете использовать команду `queue:subscribe` Artisan для регистрации URL точки (end-point), которая будет получать добавляемые в очередь задачи.

#### Регистрация подписчика push-очереди

	php artisan queue:subscribe queue_name http://foo.com/queue/receive

Теперь, когда вы войдёте в ваш профиль Iron, то увидите новую push-очередь и её URL подписки. Вы можете подписать любое число URL на одну очередь. Дальше создайте маршрут для вашей точки `queue/receive` и пусть он возвращает результат вызова метода `Queue::marshal`:

	Route::post('queue/receive', function()
	{
		return Queue::marshal();
	});

Этот метод позаботится о вызове нужного класса-обработчика задачи. Для помещения задач в push-очередь просто используйте всё тот же метод `Queue::push`, который работает и для обычных очередей.

<a name="failed-jobs"></a>
## Незаконченные задачи

Иногда вещи работают не так, как мы хотим, и запланированные задачи, бывает, аварийно завершаются из-за внутренних ошибок или некорректных входных данных. Даже лучшие из программистов допускают ошибки, это нормально. 

Laravel имеет средства для контроля над некорректным завершением задач. Если прошло заданное максимальное количество попыток запуска и задача ни разу не исполнилась до конца, завершившись исключением, она пимещается в базу данных, в таблицу `failed_jobs`. Изменить название таблицы вы можете в конфиге `app/config/queue.php`.

Данная команда создает миграцию для создания таблицы в вашей базе данных:

	php artisan queue:failed-table

Максимальное число попыток запуска задачи задается параметром `--tries` команды `queue:listen`:

	php artisan queue:listen connection-name --tries=3

Вы можете зарегистрировать слушателя события `Queue::failing`, чтобы, например, получать уведомления по e-mail, что что-то в подсистеме очередей у вас идет не так:

	Queue::failing(function($connection, $job, $data)
	{
		//
	});

Список всех незаконченных задач c их ID вам покажет команда `queue:failed`:

	php artisan queue:failed

Вы можете вручную рестартовать задачу по её ID:

	php artisan queue:retry 5

Если вы хотите удалить задачу из списка незавершенных, используйте `queue:forget`:

	php artisan queue:forget 5

Чтобы очистить весь список незавершенных задач, используйте `queue:flush`:

	php artisan queue:flush




