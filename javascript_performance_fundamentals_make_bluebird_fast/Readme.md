# Три принципа производительности в Javascript, делающие Bluebird быстрым

Компания Reaktor поделилась в своём блоге принципами и примерами оптимизации Javascript-кода, применёнными в библиотеке промисов Bluebird, созданной их сотрудником Petka Antonov (Петкой Антоновым).

[Bluebird](http://bluebirdjs.com/) — популярная JavaScript-библиотека промисов. Впервые её заметили в 2013 году, когда оказалось, что она может превосходить по скорости другие реализации промисов с похожими наборами свойств до 100 раз. Bluebird столь быстра благодаря последовательному применению некоторых основополагающих принципов оптимизации в JavaScript. В этой статье детально рассмотрены 3 наиболее важных принципа, использующихся для оптимизации Bluebird.

## 1. Минимизируйте создание функций

Создание объектов и, в частности, создание объектов функций (_примечание переводчика:_ [любая функция — объект](https://developer.mozilla.org/ru/docs/Web/JavaScript/Guide/Functions#%D0%A4%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8_%D0%B2_JavaScript)) может быть очень накладно в плане производительности, поскольку требует задействования большого количества внутренних данных. Практические реализации JavaScript содержат сборщики мусора, а значит, созданные объекты не просто так сидят в памяти — сборщик мусора постоянно ищет неиспользуемые объекты, чтобы освободить занимаемую ими память. Чем больше памяти вы используете в JavaScript, тем больше ЦПУ занимает сборка мусора и меньше остаётся для работы самого кода. В JavaScript функции являются [объектами первого класса](https://ru.wikipedia.org/wiki/%D0%9E%D0%B1%D1%8A%D0%B5%D0%BA%D1%82_%D0%BF%D0%B5%D1%80%D0%B2%D0%BE%D0%B3%D0%BE_%D0%BA%D0%BB%D0%B0%D1%81%D1%81%D0%B0). Это означает, что они имеют те же особенности и свойства, что и любые другие объекты. Если у вас есть функция, содержащая объявление другой функции (или функций), то при каждом вызове изначальной функции будут создаваться новые уникальные функции, делающие одно и то же. Рассмотрим простой пример:

```js
function trim(string) {
	function trimStart(string) {
		return string.replace(/^\s+/g, "");
	}

	function trimEnd(string) {
		return string.replace(/\s+$/g, "");
	}

	return trimEnd(trimStart(string))
}
```

Всякий раз, когда вызывается `trim`, создаются два объекта функций, представляющие `trimStart` и `trimEnd`. Но они не нужны, поскольку в них не используется ни уникальное поведение объектов, вроде присвоения свойств, ни замыкание над переменными. Единственное, зачем они используются — для функциональности содержащегося в них кода.

Этот пример легко оптимизировать — нужно лишь вынести функции из `trim`. Поскольку пример содержится в модуле, а модуль загружается в программе один раз, то для функций существует лишь одно представление:

```js
function trimStart(string) {
	return string.replace(/^\s+/g, "");
}

function trimEnd(string) {
	return string.replace(/\s+$/g, "");
}

function trim(string) {
	return trimEnd(trimStart(string))
}
```

Однако чаще всего функции выглядят как необходимое зло, от которого просто так не избавишься. Например, практически всякий раз, когда вы передаёте функцию-колбэк для отложенного вызова, колбэку для работы требуется уникальный контекст. Как правило, контекст реализуется в простой и интуитивной, но неэффективной манере — через использование замыканий. В качестве простого примера можно привести чтение файла из JSON в node при помощи стандартного асинхронного интерфейса колбэков.

```js
var fs = require('fs');

function readFileAsJson(fileName, callback) {
	fs.readFile(fileName, 'utf8', function(error, result) {
		// Новый объект функции создаётся при каждом вызове readFileAsJson.
		// Поскольку это замыкание, создаётся внутренний объект Context 
		// для состояния замыкания.
		if (error) {
			return callback(error);
		}
		// Блок try-catch необходим для обработки 
		// возможной синтаксической ошибки из-за невалидного JSON
		try {
			var json = JSON.parse(result);
			callback(null, json);
		} catch (e) {
			callback(e);
		}
	})
}
```

В этом примере колбэк, передаваемый в `fs.readFile`, нельзя вынести из `readFileAsJson`, поскольку он создаёт замыкание вокруг уникальной переменной `callback`. Следует заметить, что попытка вынести анонимный колбэк в именованную функцию ни к чему не приведёт.

Оптимизация, постоянно используемая внутри Bluebird — использование явного простого объекта для удержания контекстных данных. Для того, чтобы передать колбэк через множество уровней, понадобится выделить память лишь для одного такого объекта. Вместо того, чтобы на каждом уровне создавать новое замыкание, когда колбэк передаётся на следующий уровень, мы будем передавать явный простой объект дополнительным аргументом. Например, если в изначальной функции есть 5 уровней — значит, при использовании замыканий будет создано 5 функций и Context-объектов вместе с ними. В случае же с данной оптимизацией для этих целей будет создан всего лишь один объект.

Если бы можно было изменить `fs.readFile`, чтобы передавать туда объект контекста, оптимизацию можно было бы применить вот так:

```js
var fs = require('fs-modified');

function internalReadFileCallback(error, result) {
	// Модифицированный readFile вызывает callback при помощи объекта контекста, 
	// установленного в `this`, 
	// что на самом деле является изначально переданным колбэком
	if (error) {
		return this(error);
	}
	// Блок try-catch необходим для обработки 
	// возможной синтаксической ошибки из-за невалидного JSON
	try {
		var json = JSON.parse(result);
		this(null, json);
	} catch (e) {
		this(e);
	}
}

function readFileAsJson(fileName, callback) {
	// Модифицированная fs.readFile принимает объект контекста четвертым аргументом.
	// Нет необходимости создавать отдельный объект, чтобы помещать туда callback,
	// поэтому он может быть передан напрямую в качестве контекста

	fs.readFile(fileName, 'utf8', internalReadFileCallback, callback);
}
```

Разумеется, вы должны контролировать обе части API — без поддержки параметра контекста такая оптимизация неприменима. Однако там, где это используется (например, когда вы контролируете множество внутренних уровней), выигрыш в производительности значителен. Малоизвестный факт: некоторые встроенные JavaScript Array API, например `Array.prototype.forEach`, принимают вторым параметром объект контекста.

## 2. Минимизируйте размер объекта

Критически важно минимизировать размеры часто создаваемых объектов и объектов, создаваемых в больших количествах, таких, как промисы. [Куча](https://ru.wikipedia.org/wiki/%D0%9A%D1%83%D1%87%D0%B0_(%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C)), в которой создаются объекты в большинстве реализаций JavaScript, разделена на занятые и свободные участки. Объекты меньших размеров дольше заполняют свободное пространство, чем крупные, как следствие, оставляя сборщику мусора меньше работы. Также небольшие объекты обычно содержат меньше полей, поэтому сборщику мусора проще обходить их, помечая живые и мёртвые объекты. 

Булевы и/или ограниченные числовые поля сжимаются намного сильнее при помощи [побитовых операций](https://learn.javascript.ru/bitwise-operators). Побитовые операции в JavaScript работают с 32-битными числами. В одном поле вы можете разместить 32 булевых поля, или 8 4-битных числа, или 16 булевых и 2 8-битных числа и т. п. Чтобы код оставался читабельным, каждое логическое поле должно иметь [геттер и сеттер](https://learn.javascript.ru/getters-setters), производящие нужные побитовые операции на физическом значении. Вот пример, как можно сжать одно булево поле в число (которое в дальнейшем может быть расширено для других логических полей):

```js
// Используйте 1 << 1 для второго бита, 1 << 2 для третьего и т.д.
const READONLY = 1 << 0;

class File {
	constructor() {
		this._bitField = 0;
	}

	isReadOnly() {
		// Скобки обязательны.
		return (this._bitField & READONLY) !== 0;
	}

	setReadOnly() {
		this._bitField = this._bitField | READONLY;
	}

	unsetReadOnly() {
		this._bitField = this._bitField & (~READONLY);
	}
}
```

Методы доступа настолько коротки, что, скорее всего, они будут встроены в рантайм без дополнительных вызовов функции.

_Примечание переводчика:_ базовые сведения о работе JavaScript-компилятора, понятия об инлайн-кэшировании и встраивании функций — в статье [Прошлое и будущее компиляции JavaScript](https://habrahabr.ru/post/182802/). О работе оптимизатора — [Optimization killers](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers) (временами обновляемый оригинал) Петки Антонова и перевод [Убийцы оптимизации](http://frontender.info/optimization-killers/) (опубликован в 2014).

Два или более полей, не использующихся одновременно, можно сжать в одно поле с использованием флага, отслеживающего тип помещённого в поле значения. Однако этот способ будет экономить место только когда флаг реализован как сжатое число, как показано выше.

В Bluebird такой трюк применяется, чтобы сохранять значение выполненного промиса или причину отказа. Для этого нету отдельного поля: если промис выполнен, результат выполнения сохраняется в поле для колбэка отказа, если промис отклонён, то причину отказа можно хранить в поле колбэка успешного выполнения. Опять же, обращения к значениям следует делать через функции доступа, скрывающие детали реализации.

Если объект должен хранить список сущностей, вы можете избежать создания отдельного массива, сохраняя значения напрямую в индексированные свойства объекта. Вместо того, чтобы писать:

```js
class EventEmitter {
	constructor() {
		this.listeners = [];
	}

	addListener(fn) {
		this.listeners.push(fn);
	}
}
```

Вы можете избежать массива:

```js
class EventEmitter {
	constructor() {
		this.length = 0;
	}

	addListener(fn) {
		var index = this.length;
		this.length++;
		this[index] = fn;
	}
}
```

Если поле `.length` можно ограничить малым числом (например, 10-битным, т.е. `event emitter` сможет иметь максимум 1024 слушателей), его можно сделать частью побитового поля, содержащего другие ограниченные числа и булевы значения.

## 3. Используйте [no-op](https://ru.wikipedia.org/wiki/NOP) функции. Перезаписывайте их лениво для реализации дорогостоящих опциональных функций

Bluebird содержит несколько опциональных функций, вызывающих равномерную потерю производительности всей библиотеки при использовании. Это ворнинги, трассировки стека, возможность отмены, Promise.prototype.bind, мониторинг состояний промисов и т.п. Эти функции требуют вызовов функций-перехватчиков по всей библиотеке. Например, функция мониторинга промисов требует вызова перехватчика каждый раз при создании промиса.

Гораздо проще проверить перед вызовом, включена ли функция мониторинга, чем запускать её каждый раз вне зависимости от фактического состояния. Однако, благодаря инлайн-кэшам (_примечание переводчика:_ [вот ещё одна заметка](https://gist.github.com/twokul/9501770) по теме) и встраиванию функций, эта операция может быть полностью упрощена для пользователей, у которых отключен мониторинг. Для этого присвоим изначальному методу-перехватчику пустую функцию:

```js
class Promise {

	// ...

	constructor(executor) {
		// ...
		this._promiseCreatedHook();
	}

	// Просто пустой no-op метод.
	_promiseCreatedHook() {}

}
```

Теперь, если пользователь не включил функцию мониторинга, оптимизатор видит, что перехватчик ничего не делает, и упрощает его. Получается, что метод-перехватчик в конструкторе попросту не существует.

Чтобы функция мониторинга работала, её включение должно перезаписывать все связанные no-op функции на их реальную имплементацию:

```js
function enableMonitoringFeature() {
	Promise.prototype._promiseCreatedHook = function() {
		// Реальная имплементация
	};

	// ...
}
```

Такая перезапись метода аннулирует инлайн-кэши, созданные для объектов класса Promise. Это следует сделать при старте приложения, перед тем, как будут созданы любые промисы. Таким образом инлайн-кэши, сделанные после этого, не будут знать, что существовали no-op функции.

---

_Оригинал: [Three JavaScript performance fundamentals that make Bluebird fast](https://reaktor.com/blog/javascript-performance-fundamentals-make-bluebird-fast/), автор: Petka Antonov._

_Перевод: [Андрей Алексеев](https://github.com/aalexeev239/), редактура: @jabher._
