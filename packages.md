git d4ab3b757eb93c27fd2f159bc3fad88dd57d9eff

---

# Разработка пакетов

- [Введение](#introduction)
- [Создание пакета](#creating-a-package)
- [Структура пакетов](#package-structure)
- [Сервис-провайдеры](#service-providers)
- [Отложенные сервис-провайдеры](#deferred-providers)
- [Соглашения](#package-conventions)
- [Процесс разработки](#development-workflow)
- [Маршрутизация в пакетах](#package-routing)
- [Конфиги в пакетах](#package-configuration)
- [Шаблоны пакетов](#package-views)
- [Миграции пакетов](#package-migrations)
- [Статические файлы пакетов](#package-assets)
- [Публикация пакетов](#publishing-packages)

<a name="introduction"></a>
## Введение

Пакеты (packages) - основной способ добавления нового функционала в Laravel. Пакеты могут быть всем, чем угодно - от классов для удобной работы с датами типа [Carbon](https://github.com/briannesbitt/Carbon) до целых библиотек BDD-тестирования наподобие [Behat](https://github.com/Behat/Behat).

Конечно, всё это разные типы пакетов. Некоторые пакеты самостоятельны, что позволяет им работать в составе любого фреймворка, а не только Laravel. Примерами таких отдельных пакетов являются Carbon и Behat. Любой из них может быть использован в Laravel после простого указания его в файле `composer.json`.

С другой стороны, некоторые пакеты разработаны специально для использования в Laravel. В предыдущей версии Laravel такие пакеты назывались "bundles". Они могли содержать роуты, контроллеры, шаблоны, настройки и миграции, специально рассчитанные для улучшения приложения на Laravel. Так как для разработки самостоятельных пакетов нет особенных правил, этот раздел документации в основном посвящён разработке именно пакетов для Laravel.

Все пакеты Laravel распространяются через [Packagist](http://packagist.org) и [Composer](http://getcomposer.org), поэтому нужно изучить эти прекрасные средства распространения кода для PHP.

<a name="creating-a-package"></a>
## Создание пакета

Простейший способ создать пакет для использования в Laravel - с помощью команды `workbench` инструмента командной строки Artisan. Сперва вам нужно установить несколько параметров в файле `app/config/workbench.php`. Там вы найдёте такие настройки `name` и `email`. Их значения будут использованы при генерации `composer.json` для вашего нового пакета. Когда вы заполнили эти значения, то всё готово для создания заготовки.

#### Использование команды Workbench

	php artisan workbench vendor/package --resources

Имя поставщика (vendor) - способ отличить ваши собственные пакеты от пакетов других разработчиков. К примеру, если я, Taylor Otwell (автор Laravel - прим. пер.), хочу создать новый пакет под названием "Zapper", то имя поставщика может быть `Taylor`, а имя пакета - `Zapper`. По умолчанию команда `workbench` сгенерирует заготовку в общепринятом стиле пакетов, однако команда `resources` может использоваться для генерации специфичных для Laravel папок, таких как `migrations`, `views`, `config` и прочих.

Когда команда `workbench` была выполнена, ваш пакет будет доступен в папке `workbench` текущей установки Laravel. Дальше вам нужно зарегистрировать сервис-провайдер пакета, который создан автоматически предыдущей командой. Это можно сделать, добавив его к массиву `providers` файла `app/config/app.php`. Это укажет Laravel, что пакет должен быть загружен при запуске приложения. Имена классов поставщики услуг следуют схеме `[Package]ServiceProvider`. Таким образом, в примере выше мы должны были бы добавить `Taylor\Zapper\ZapperServiceProvider` к массиву `providers`.

Как только поставщик зарегистрирован вы готовы к началу разработки. Однако, перед этим, рекомендуется ознакомиться с материалом ниже, чтобы узнать о структуре пакетов и процессом их разработки.

> **Примечание:** если ваш сервис-провайдер не может быть найден, выполните команду `php artisan dump-autoload` из корня вашего приложения.

<a name="package-structure"></a>
## Структура пакетов

При использовании команды `workbench` ваш пакет будет настроен согласно общепринятым нормам, что позволит ему успешно интегрироваться с другими частями Laravel.

#### Базовая структура папок внутри пакета

	/src
		/Vendor
			/Package
				PackageServiceProvider.php
		/config
		/lang
		/migrations
		/views
	/tests
	/public

Давайте познакомимся с этой структурой поближе. Папка `src/Vendor/Package` - хранилище всех классов вашего пакета, включая `ServiceProvider`. Папки `config`, `lang`, `migrations` и `views` содержат соответствующие ресурсы для вашего пакета (настройки, файлы локализации, миграции и шаблоны). 

<a name="service-providers"></a>
## Сервис-провайдеры

Сервис-провайдеры - классы первичной инициализации пакета. По умолчанию они могут содержать два метода: `boot` и `register`. Внутри этих методов вы можете выполнять любой код - подключать файл с роутами, регистрировать ключи в контейнере IoC, устанавливать обработчики событий или что-то ещё.

Метод `register` вызывается сразу после регистрации сервис-провайдером, тогда как команда `boot` вызывается только перед тем, как будет обработан запрос. Таким образом, если вашему сервис-провайдеру требуется другой сервис-провайдер, который уже был зарегистрирован, или вы перекрываете другой сервис-провайдер - вам нужно использовать метод `boot`.

При создании пакета с помощью команды `workbench`, метод `boot` уже будет содержать одно действие:

	$this->package('vendor/package');

Этот метод позволяет Laravel определить, как правильно загружать шаблоны, настройки и другие ресурсы вашего приложения. Обычно вам не требуется изменять эту строку, так как она настраивает пакет в соответствии с обычными нормами.

По умолчанию, после регистрации пакета `vendor/package`, его ресурсы доступны по имени `package`. Вы можете изменить это поведение:

	// Хочу вместо `package` обращаться к ресурсам пакета по имени `custom-namespace`
	$this->package('vendor/package', 'custom-namespace');

	// Загрузить шаблон пакета по новому имени
	$view = View::make('custom-namespace::foo');

Если вы хотите изменить папки, где располагаются ресурсы пакета (папки `config`, `lang`, `migrations`, `views`), вы можете указать путь до этого места третьим аргументом:

	$this->package('vendor/package', null, '/path/to/resources');

<a name="deferred-providers"></a>
## Отложенные сервис-провайдеры	

Если вы пишете сервис-провайдер, который не регистрирует ни один из ресурсов, то вы можете сделать его "отложенным". Такие сервис-провайдеры загружаются и регистрируются только тогда, когда сервисы, их использующие, "достаются"" из IoC-контейнера. 

Для того, чтобы сделать сервис-провайдер отложенным, установите свойство `$defer` в `true`:

	protected $defer = true;

Далее, вам нужно указать, каким объектам из IoC-контейнера нужен этот сервис-провайдер. Для этого создайте метод `provides`, который возвращает массив ключей, под которыми объекты регистрируются в IoC-контейнере. Например, ваш сервис-провайдер регистрирует в IoC-контейнер объекты с именами `package.service` и `package.another-service`. Соответственно, метод должен выглядеть так:

	public function provides()
	{
		return array('package.service', 'package.another-service');
	}

<a name="package-conventions"></a>
## Соглашения

При использовании ресурсов из пакета (конфиги или шаблоны) для отделения имени пакета обычно используют двойное двоеточие (::).

#### Загрузка шаблона из пакета

	return View::make('package::view.name');

#### Чтение параметров конфига в пакете

	return Config::get('package::group.option');

> **Примечание:** если ваш пакет содержит миграции, подумайте о том, чтобы сделать префикс к имени миграции - для предотвращения возможных конфликтов с именами классов в других пакетах.

<a name="development-workflow"></a>
## Процесс разработки

При разработке пакета как правило удобно работать в контексте приложения. Для начала сделайте чистую установку Laravel, затем используйте команду `workbench` для создания структуры пакета.

После того как пакет создан, вы можете сделать `git init` изнутри папки `workbench/[vendor]/[package]`, а затем - `git push` для отправки пакета напрямую в хранилище (например, github). Это позвоит вам удобно работать в среде приложения без необходимости постоянно выполнять команду `composer update`.

Теперь, как ваши пакеты расположены в папке `workbench`, у вас может возникнуть вопрос: как Composer знает, каким образом загружать эти пакеты? Laravel автоматически сканирует папку `workbench` на наличие пакетов, загружая их файлы при запуске приложения.

Если вам нужно зарегистрировать файлы автозагрузки вашего пакета, можно использовать команду `php artisan dump-autoload`. Эта команда пересоздаст файлы автозагрузки для корневого приложения, а также всех пакетов в `workbench`, которые вы успели создать.

#### Выполнение команды автозагрузки

	php artisan dump-autoload

<a name="package-routing"></a>
## Маршрутизация в пакетах

В предыдущей версии Laravel для указания URL, принадлежащих пакету, использовался параметр `handles`. Начиная с Laravel 4 пакеты могут обрабатывать любой URI. Для загрузки файлов с роутами просто подключите его через `include` из метода `boot` сервис-провайдера пакета.

#### Подключение файла с маршрутами из сервис-провайдера

	public function boot()
	{
		$this->package('vendor/package');

		include __DIR__.'/../../routes.php';
	}

> **Примечание:** Если ваш сервис-провайдер использует контроллеры, вам нужно убедиться, что они верно настроены в секции автозагрузки вашего файла `composer.json`.

<a name="package-configuration"></a>
## Конфиги в пакетах

Некоторые пакеты могут требовать файлов настройки (config). Эти файлы должны быть определены аналогично файлам настроек обычного приложения. И затем, при использовании стандартной команды для регистрации ресурсов пакета `$this->package`, они будут доступны через обычный синтаксис с двойным двоеточием (::).

#### Чтение конфига пакета

	Config::get('package::file.option');

Однако если ваш пакет содержит всего один файл с настройками, вы можете назвать его `config.php`. Когда это сделано, то его параметры можно становятся доступными напрямую, без указания имени файла.

#### Чтение конфига пакета

	Config::get('package::option');

Иногда вам может быть нужно зарегистрировать ресурсы пакета вне обычного вызова `$this->package`. Обычно это требуется, если ресурс расположен не в стандартном месте. Для регистрации ресурса вручную вы можете использовать методы `addNamespace` классов `View`, `Lang` и `Config`.

#### Ручная регистрация пространства имён ресурсов

	View::addNamespace('package', __DIR__.'/path/to/views');

Как только пространство имён было зарегистрировано, вы можете использовать его имя и двойное двоеточие для получения доступа к ресурсам:

	return View::make('package::view.name');

Параметры методов `addNamespace` одинаковы для классов `View`, `Lang` и `Config`.

### Редактирование конфигов пакета

Когда другие разработчики устанавливают ваш пакет им может потребоваться изменить некоторые из настроек. Однако если они сделают это напрямую в коде вашего пакета, изменения будут потеряны при следующем обновлении пакета через Composer. Вместо этого нужно использовать команд `config:publish` интерфейса Artisan.

#### Выполнение команды публикации настроек

	php artisan config:publish vendor/package

Эта команда скопирует файлы настроек вашего приложения в папку `app/config/packages/vendor/package`, где разработчик может их безопасно изменять.

> **Примечание:** Разработчик также может создать настройки, специфичные для каждой среды, поместив их в `app/config/packages/vendor/package/environment`.

<a name="package-migrations"></a>
## Миграции пакетов

Вы можете легко создавать и выполнять [миграции](docs/migrations) для любого из ваших пакетов. Для создания миграции в `workbench` используется команда `--bench` option:

#### Создание миграций для пакета в `workbench`

	php artisan migrate:make create_users_table --bench="vendor/package"

#### Выполнение миграций пакета в `workbench`

	php artisan migrate --bench="vendor/package"

Для выполнения миграции для законченного пакета, который был установлен через Composer в папку `vendor`, вы можете использовать ключ `--package`:

#### Выполнение миграций установленного пакета:

	php artisan migrate --package="vendor/package"

<a name="package-assets"></a>
## Статические файлы пакетов

Некоторые пакеты могут содержать внешние ресурсы, такие как JavaScript-код, CSS и изображения. Однако мы не можем обращаться к ним напрямую через папки `vendor` и `workbench`, так как они находятся ниже DOCROOT сервера, поэтому нам нужно перенести их в папку `public` нашего приложения. Это делает команда `asset:publish`.

#### Перемещение ресурсов пакета в папку `public`

	php artisan asset:publish

	php artisan asset:publish vendor/package

Если пакет находится в `workbench`, используйте ключ `--bench`:

	php artisan asset:publish --bench="vendor/package"

Эта команда переместит ресурсы в `public/packages` в соответствии с именем поставщика и пакета. Таким образом, внешние ресурсы пакета `userscape/kudos` будут помещены в папку `public/packages/userscape/kudos`. Соблюдение этого соглашения о путях позволит вам использовать их в HTML-коде шаблонов вашего пакета.

<a name="publishing-packages"></a>
## Публикация пакетов

Когда ваш пакет готов, вы можете отправить его в [Packagist](http://packagist.org). Если пакет предназначен для Laravel рекомендуется добавить тег `laravel` в файл `composer.json` вашего пакета.

Также обратите внимание, что полезно присваивать вашим выпускам номера (tags), позволяя другим разработчикам использовать стабильные версии в их файлах `composer.json`. Если стабильный выпуск ещё не готов, можно добавить директиву Composer `branch-alias`.

Когда ваш пакет опубликован, вы можете свободно продолжать его разработку в среде вашего приложения, созданной командой `workbench`. Это отличный способ, позволяющий удобно продолжать над ним работу даже после публикации.

Некоторые организации создают собственные частные хранилища пакетов для своих сотрудников. Если вам это требуется, изучите документацию проекта [Satis](http://github.com/composer/satis), созданного командой разработчиков Composer.
