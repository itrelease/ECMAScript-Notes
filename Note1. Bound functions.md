# Заметка №1. ECMAScript. Bound functions.

Как мы уже знаем, в редакции `ECMA-262-5` был стандартизирован метод `bind`. Источником для текущей версии послужил одноименный метод из библиотеки [Prototype.js](http://www.prototypejs.org/), но имеющий некоторые отличия. Основное предназначение хорошо известно и удачно используется в `ES3`. Главная задача данной заметки — показать детали и разницу в реализации `ES5`.

#### Цели.
`Function.prototype.bind` имеет два предназначения: статическое связывание (`bound`) значения `this` функции и частичное применение (`partial application`).

#### Связывание значения this.
Основным предназначением метода `bind` является статическое связывание значения `this` функции в последующих вызовах. Как мы уже знаем, значение `this` может изменяться при различных вызовах функции. Т.о. главной целью `bind` является зафиксировать значение `this`. Это может пригодиться, когда мы хотим использовать метод объекта в качестве функции обработчика (`event handler`) какого-либо события.

    var widget = {
        state: {...},
        onClick: function onWidgetClick(event) {
            if (this.state.active) {
                ...
            }
        }
    };

    document.getElementById('widget').onclick = widget.onClick.bind(widget);

Для простоты, обработчик устанавливается присваиванием атрибуту `onclick`, в реальных же условиях, лучше воспользоваться методом `addEventListener` или `attachEvent`.  В примере показано как можно выставить значение `this`, чтобы при вызове обработчика оно указывало на объект `widget`.

#### Частичное применение.
Метод `bind` также может использоваться для каррирования ([currying](http://ru.wikipedia.org/wiki/%D0%9A%D0%B0%D1%80%D1%80%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5), в математике это называется _частичным применением_ функции). Это техника преобразования функции от множества аргументов в цепочку вложенных функций с единственным аргументом. Это означает, что мы можем создать функцию, основанную на других функциях и некоторые из аргументов этой новой функции могут быть предустановлены. Применение такой функции с частью аргументов называется _частичным применением_ функции.

    function foo(x, y) {
       //partial
       if (typeof y == 'undefined') {
           return function partialFoo(y) {
               //complete
               return x + y;
           };
       }
       // complete
       return x + y;
    }

    foo(10, 20); // 30
    foo(10)(20); // 30;

    var partialFoo = foo(10); // function
    partialFoo(20); // 30

На практике _частичное применение_ функции может пригодиться, когда необходимо часто вызывать функцию в которой одно из значений аргумента повторяется. Также иногда бывает необходимо заранее передать в функцию обработчик (`event handler`) некоторые данные. Тема _частичного применения_ функции относится к некоторым математическим теоремам и к [лямбда-исчислению](http://ru.wikipedia.org/wiki/%D0%9B%D1%8F%D0%BC%D0%B1%D0%B4%D0%B0-%D0%B8%D1%81%D1%87%D0%B8%D1%81%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5) где функции могут принимать только один аргумент.

    function foo(x, y) {
        return x + y;
    }

    var partialFoo = foo.bind(null, 10);
    partialFoo(20); // 30

В последнем примере значение `this` для нас не играет никакой роли, так как в функции мы его не используем.

#### Реализация.
Реализация метода `bind` в `ES5` отличается от версии, которую используют в `ES3`. В функциях, которые получаются путем использования метода `bind`, отсутствуют некоторые внутренние св-ва, которые присутствуют у обычных функций. Это сделано для оптимизации. Такие функции не имеют таких свойств как: `prototype`, `[[Code]]` (тело функции — код), `[[FormalParameters]]` (список формальных параметров функции), а также `[[Scope]]` (лексическая область видимости, в которой была создана функция).

В то же время такие функции имеют ряд дополнительных внутренних свойств: `[[TargetFunction]]` — ссылка на оригинальную функцию; `[[BoundThis]]` — связанное значение `this`; `[[BoundArgs]]` — связанные аргументы, которые используются в `partial function`.

Три внутренних метода у bound-функции — `[[Call]]`, `[[Construct]]` и `[[HasInstance]]` переопределены.

Внутренний метод  `[[Call]]` функции вызывается каждый раз, когда функция активируется (к примеру, вызов функции `foo()` или когда к функции применяются методы `call` и `apply`).

После того, как внутренний метод `[[Call]]`  bound-функции отработает, вызывается оригинальный метод `[[Call]]` передавая нужное значение `this`. Это можно представить на псевдо-коде так:

    boundFn.[[Call]] = function(thisValue, extraArgs):
        var boundArgs = boundFn.[[BoundArgs]],
            boundThis = boundFn.[[BoundThis]],
            targetFn = boundFn.[[TargetFunction]],
            args = boundArgs.concat(extraArgs);

        return targetFn.[[Call]](boundThis, args);

Далее вызывается стандартный метод функции `[[Call]]`, который устанавливает контекст исполнения для свойства `[[Code]]` (см. [раздел 13.2.1 — [[Call]]](http://es5.github.com/#x13.2.1])).

    function F(x, y) {
        this.x = x;
        this.y = y;
        return this;
    }

    // связываем значение "this" и передаем аргумент "x"
    var BoundF = F.bind({z: 30}, 10);

    // создаем объект и передаем аргумент "y"
    var objectFromCall = boundF(20);

    // т.к. значение "this" внутри BoundF указывает на
    // объект {z: 30} мы получаем:
    alert([
        objectFromCall.x, // 10, из [[BoundArgs]]
        objectFromCall.y, // 20, из extraArgs
        objectFromCall.z  // 30, из [[BoundThis]]
    ]);

Если внимательно посмотреть на псевдо-код работы `[[Call]]` метода bound-функции, то можно заметить, что значение `this` (первый аргумент `thisValue`) нигде не участвует, а только `[[BoundThis]]` будет передаваться в оригинальный `[[Call]]`. Таким образом, даже вызывая функцию с использованием методов `call` и `apply` значение `this` не изменится:

    var b = BoundF.call({z: 40});
    alert(b.z); // по-прежнему 30

Перегруженный метод `[[Construct]]` ([15.3.4.5.2](http://es5.github.com/#x15.3.4.5.2)) после своего выполнения также передаст управление оригинальному методу `[[Construct]]`, который описан в секции [13.2.2](http://es5.github.com/#x13.2.2). Есть одно важное замечание при использовании bound-функции в качестве конструктора. Оригинальный метод `[[Construct]]` вызывается, после того как отработает перегруженный метод `[[Construct]]`, далее активируется **оригинальный** метод `[[Call]]`, а не перегруженный. Это означает, что в качестве значения `this` будет использоваться **только что созданный объект**, а не _связанное значение_.

Это довольно логично, но некоторым может показаться странным такое поведение. Это важное отличие от реализации метода `bind` на JavaScript. Так метод `bind` в библиотеке `Prototype.js` всегда использует связанное значение `this`. Т.о. если в конструкторе явно возвращается значение `this` то мы получим связанное значение. В противном случае результат bound-функции конструктора неопределен и зависит от возвращаемого значения bound-функции.

По большому счету, невозможно реализовать на JavaScript метод `bind`, который будет полностью удовлетворять реализации `ES5`. Но можно добиться [приемлемой эмуляции](http://code.google.com/p/js-examples/source/browse/trunk/bind_emulation.js).

В реализации `bind` метода в редакции `ES5`, мы получим следующие результат:

    // добавим св-во в прототип объекта
    F.prototype.a = 40;

    // 15.3.4.5.2 [[Construct]]  для bound-функции
    var objectFromConstruct = new BoundF(20);

    alert([
        objectFromConstruct.x, // 10, из [[BoundArgs]]
        objectFromConstruct.y, // 20, из extraArgs
        objectFromConstruct.z, // undefined, т.к. не используется [[BoundThis]]
        objectFromConstruct.a  // 40, из прототипа F.prototype === objectFromConstruct.[[Prototype]]
    ]); 

Как видно из примера у bound-функции отсутствует св-во `prototype` и значение берется из прототипа оригинальной функции. Даже если мы вручную добавим это св-во в bound-функцию, ничего не изменится, т.к. `[[Construct]]` и `[[Call]]` используют внутри оригинальную функцию.

    alert(BoundF.prototype); // undefined

    BoundF.prototype = {
        constructor: BoundF,
        a: 100,
        b: 200
    };

    var objectFromConstruct2 = new BoundF;
    // св-ва по прежнему берутся из прототипа оригинальной функции
    alert([
        objectFromConstruct2.a // 40, из F.prototype а не 100
        objectFromConstruct2.b // undefined
    ]);

Резюмируя:

* Простой вызов — `BoundF()` — используется `boundThis`;
* `call/apply` — `BoundF.call(manualThis) — используется `boundThis`;
* Вызов в качестве конструктора — `new BoundF()` — используется только что созданный объект;

Перегруженный метод `[[HasInstance]]` ([15.3.4.5.3](http://es5.github.com/#x15.3.4.5.3)) после своего выполнения передает управление оригинальному методу `[[HasInstance]]` ([15.3.5.3](http://es5.github.com/#x15.3.5.3)). Этот внутренний метод используется оператором `instanceof`, который возвращает `true` как для оригинальной так и для bound функций:

    alert([
        objectFromConstruct instanceof F, // true
        objectFromConstruct instanceof BoundF // true
    ]);

Нужно отметить, что из-за такой делегации оператор `instanceof` вернет `true` даже для того объекта, который был создан через конструктор оригинальной функции:

    var object = new F;

    alert([
        object instanceof F, // true
        object instanceof BoundF // true
    ]);

Это также относится к использованию метода `Object.create`. Даже если мы определим св-во `prototype` у `BoundF` и передадим его в качестве первого аргумента в `Object.create`, оператор `instanceof` вернет `false` как для оригинальной функции конструктора, так и для bound-функции конструктора:

    var boundProto = {
        constructor: BoundF
    };

    BoundF.prototype = boundProto;

    var foo = Object.create(BoundF.prototype);
    console.log(Object.getPrototypeOf(foo) === BoundF.prototype); // true

    // из-за перегруженного [[HasInstance]],
    // который в итоге вызовет F.[[HasInstance]] мы получим false
    console.log(foo instanceof BoundF); // false
    console.log(foo instanceof F); // false

    var bar = Object.create(F.prototype);
    console.log(bar instanceof BoundF); //true
    console.log(bar instanceof F); // true

    // но если мы установим F.prototype
    // в нужный нам объект, то instanceof
    // вернет true
    F.prototype = boundProto;
    console.log(foo instanceof BoundF); // true
    console.log(foo instanceof F); // true

Есть и еще одна особенность у bound-функций. Если у такой функции попытаться запросить св-во `caller` или `arguments` то немедленно будет брошено исключение `TypeError` даже в `non-strict` режиме, в отличие от обычных функций:

    alert([
        F.caller, // ошибка только в strict режиме
        F.arguments, // ошибка только в strict режиме

        BoundF.caller, // всегда ошибка
        BoundF.arguments // всегда ошибка
    ]);

#### Функция конструктор с различным количеством аргументов.
На [MDC](https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/Function/bind#Supplemental) была описана интересная техника использования функции конструктора с различным количеством аргументов. Для этого в конструктор, в качестве аргумента передается массив с нужными значениями. Получается некоторое подобие `Function.prototype.apply`, но с возможностью использования с ключевым словом `new` — для использования функции как конструктор. Ниже представлена немного модифицированная версия примера с MDC:

    Function.prototype.construct = function(args) {
        var boundArgs = [].concat.apply(null, args),
            boundFn = this.bind.apply(this, boundArgs);

        return new boundFn();
    };

    function Point(x, y) {
        this.x = x;
        this.y = y;
    }

    var point = Point.construct([2, 4]);
    console.log(point.x, point.y) // 2, 4

Но нужно помнить, что такой подход не очень эффективен, так как при каждом вызове создается `bound`-функция.