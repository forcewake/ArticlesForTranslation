
[Source](http://habrahabr.ru/post/201832/ "Permalink to Часть 1 — Scope и Digest / Хабрахабр")

# Часть 1 — Scope и Digest / Хабрахабр

Angular — зрелый и мощный JavaScript-фреймворк. Он довольно большой и основан на множестве новых концепций, которые необходимо освоить, чтобы работать с ним эффективно. Большинство разработчиков, знакомясь с Angular, сталкиваются с одними и теми же трудностями. Что конкретно делает функция digest? Какие существуют способы создания директив? Чем отличается сервис от провайдера?

Несмотря на то, что у Angular довольно хорошая [документация][1], и существует куча [сторонних ресурсов][2], нет лучшего способа изучить технологию, чем разобрать ее по кусочкам и вскрыть ее магию.

В этой серии статей я **собираюсь воссоздать AngularJS с нуля**. Мы сделаем это вместе шаг за шагом, в процессе чего, вы намного глубже поймете внутреннее устройство Angular.

В первой части этой серии мы рассмотрим устройство областей видимости (scope), и то, как, на самом деле, работают **$eval**, **$digest** и **$apply**. Проверка данных на изменение (dirty-checking) в Angular кажется магией, но это не так — вы все увидите сами.

#### Подготовка


[Исходники проекта][3] доступны на github, но я бы не советовал вам их просто копировать себе. Вместо этого, я настаиваю на том, чтобы вы сделали все сами, шаг за шагом, поигравшись с кодом и покопавшись в нем. В тексте я использую [JS Bin][4], так что вы можете прорабатывать код, даже не покидая страницу (**прим. пер.** — в переводе будут только ссылки на JS Bin код).

Мы будем использовать [Lo-Dash][5] для некоторых низкоуровневых операций с массивами и объектами. Сам Angular не использует Lo-Dash, но для наших целей имеет смысл убрать как можно больше шаблонного низкоуровневого кода. Везде, где вы встретите в коде (**_**) (символ подчеркивания) — вызываются функции Lo-Dash.

Так же мы будем использовать функцию **console.assert** для простейших проверок. Она должна быть доступна во всех современных JavaScript-окружениях.

Вот пример Lo-Dash и assert в действии:

[Код на JS Bin][6]


**Просмотреть код**


    var a = [1, 2, 3];
    var b = [1, 2, 3];
    var c = _.map(a, function(i) {
      return i * 2;
    });

    console.assert(a !== b);
    console.assert(_.isEqual(a, b));
    console.assert(_.isEqual([2, 4, 6], c));



Консоль:


    true
    true
    true







#### Объекты — область видимости (scope-ы)


Объекты области видимости в Angular — это обычные JavaScript-объекты, к которым можно добавлять свойства стандартным способом. Они создаются при помощи конструктора **Scope**. Давайте напишем простейшую его реализацию:


    function Scope() {
    }



Теперь при помощи оператора **new** можно создавать scope-объекты и добавлять в них свойства.


    var aScope = new Scope();
    aScope.firstName = 'Jane';
    aScope.lastName = 'Smith';



В этих свойствах нет ничего особенного. Не нужно назначать никаких специальных сеттеров (setters), нет никаких ограничений на типы значений. Вместо этого вся магия заключена в двух функциях: **$watch** и **$digest**.

#### Наблюдение за свойствами объекта: $watch и $digest


**$watch** и **$digest ** — две стороны одной медали. Вместе они образуют ядро того, чем в Angular являются scope-объекты: реакцию на изменение данных.

Используя **$watch** можно добавить в scope "наблюдателя". Наблюдатель — это то, что будет получать уведомление, когда в соответствующем scope произойдет изменение.

Создается наблюдатель передачей в **$watch** двух функций:

* _watch-функция_, которая возвращает те данные, изменение которых вам интересно
* _listener-функция_, которая будет вызываться при изменении этих данных



&gt; В Angular вы вместо watch-функции обычно использовали watch-выражение. Это строка (что-то типа «user.firstName»), которую вы указывали в html при связывании, как атрибут директивы, или напрямую из JavaScript. Эта строка разбиралась и компилировалась Angular-ом в, аналогичную нашей, watch-функцию. Мы рассмотрим, как это делается в следующей статье. В этой же статье будем придерживаться низкоуровневого подхода, используя watch-функции.


Для реализации **$watch**, нам необходимо где-то хранить все регистрируемые наблюдатели. Давайте добавим для них массив в конструктор **Scope**:


    function Scope() {
      this.$$watchers = [];
    }



Префикс **$$** обозначает то, что переменная является приватной во фреймворке Angular и не должна вызываться из кода приложения.

Сейчас уже можно определить функцию **$watch**. Она будет принимать две функции в качестве аргументов и сохранять их в массиве **$$watchers**. Предполагается, что эта функция нужна каждому scope-объекту, поэтому давайте вынесем ее в прототип:


    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn
      };
      this.$$watchers.push(watcher);
    };



Обратной стороной медали является функция **$digest**. Она запускает всех наблюдателей, зарегистрированных для данной области видимости. Давайте опишем ее простейшую реализацию, в которой просто перебираются все наблюдатели, и у каждого из них вызывается listener-функция:


    Scope.prototype.$digest = function() {
      _.forEach(this.$$watchers, function(watch) {
        watch.listenerFn();
      });
    };



Теперь можно зарегистрировать наблюдатель, запустить **$digest**, в результате чего отработает его listener-функция:

[Код на JS Bin][7]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$digest = function() {
      _.forEach(this.$$watchers, function(watch) {
        watch.listenerFn();
      });
    };

    var scope = new Scope();
    scope.$watch(
      function() {console.log('watchFn'); },
      function() {console.log('listener'); }
    );

    scope.$digest();
    scope.$digest();
    scope.$digest();



Консоль:


    "listener"
    "listener"
    "listener"





Само по себе это еще не особо полезно. Чего бы нам действительно хотелось, так это того, чтобы обработчики запускались только в том случае, если действительно изменились данные, обозначенные в watch-функции.

#### Обнаружение изменения данных


Как говорилось раньше, watch-функция наблюдателя должна возвращать данные, изменение которых ей интересны. Обычно эти данные находятся в scope, поэтому, для удобства, scope передается ей в качестве аргумента. Watch-функция, наблюдающая за **firstName** из scope будет выглядеть примерно так:


    function(scope) {
      return scope.firstName;
    }



В большинстве случаев watch-функция выглядит именно так: извлекает интересные ей данные из scope и возвращает их.

Работа функции **$digest** заключается в том, чтобы вызвать эту watch-функцию и сравнить полученное от нее значение с тем, что она возвращала в прошлый раз. Если значения различаются — то данные "грязные", и необходимо вызвать соответствующую listener-функцию.

Чтобы сделать это, **$digest** должна запоминать последнее возвращенное значение для каждой watch-функции, а так как у нас для каждого наблюдателя уже есть свой объект, удобнее всего хранить эти данные в нем. Вот новая реализация функции **$digest**, которая проверяет данные на изменение для каждого наблюдателя:


    Scope.prototype.$digest = function() {
      var self = this;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
        }
        watch.last = newValue;
      });
    };



Для каждого наблюдателя вызывается watch-функция, передавая текущий scope как аргумент. Далее полученное значение сравнивается с предыдущим, сохраненным в атрибуте **last**. Если значения различаются — вызывается listener. Для удобства, в listener в качестве аргументов передаются оба значения и scope. В конце, в **last**-атрибут наблюдателя записывается новое значение, чтобы можно было проводить сравнение в следующий раз.

Давайте посмотрим, как в этой реализации запускаются listener-ы при вызове **$digest**:

[Код на JS Bin][8]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$digest = function() {
      var self = this;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
        }
        watch.last = newValue;
      });
    };

    var scope = new Scope();
    scope.firstName = 'Joe';
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.firstName;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    // We haven't run $digest yet so counter should be untouched:
    console.assert(scope.counter === 0);

    // The first digest causes the listener to be run
    scope.$digest();
    console.assert(scope.counter === 1);

    // Further digests don't call the listener...
    scope.$digest();
    scope.$digest();
    console.assert(scope.counter === 1);

    // ... until the value that the watch function is watching changes again
    scope.firstName = 'Jane';
    scope.$digest();
    console.assert(scope.counter === 2);



Консоль:


    true
    true
    true
    true





Сейчас у нас уже реализовано ядро Angular-овсого scope: регистрация наблюдателей и запуск их в фукнции **$digest**.

Так же мы можем уже сейчас сделать пару выводов, касающихся производительности scope-ов в Angular:

* Добавление данных в scope само по себе не влияет на производительность. Если нет наблюдателей, следящих за данными, то без разницы добавлены ли данные в scope или нет. Angular не перебирает свойства scope, он перебирает наблюдателей.
* Каждая watch-функция обязательно вызывается во время работы $digest. По этой причине имеет смысл уделить внимание тому, сколько у вас наблюдателей, а так же производительности каждой watch-функции самой по себе.



#### Оповещение о том, что происходит digest


Если вам необходимо получать оповещения, о том, что выполняется **$digest**, можно воспользоваться тем фактом, что каждая watch-функция, в процессе работы $digest, обязательно запускается. Нужно просто зарегистрировать watch-функцию без listener.

Чтобы учесть это, в функции **$watch** необходимо проверять, не пропущен ли listener, и если да — подставлять вместо него функцию-заглушку:


    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { }
      };
      this.$$watchers.push(watcher);
    };



Если вы используете этот шаблон, имейте ввиду, Angular учитывает возвращаемое из watch-функции значение, даже если listener-функция не объявлена. Если вы будете возвращать какое-либо значение, оно будет участвовать в проверке на изменения. Чтобы не быть причиной лишней работы — просто не возвращайте ничего из функции, по умолчанию всегда будет возвращаться **undefined**:

[Код на JS Bin][9]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { }
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$digest = function() {
      var self = this;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
        }
        watch.last = newValue;
      });
    };

    var scope = new Scope();

    scope.$watch(function() {
      console.log('digest listener fired');
    });

    scope.$digest();
    scope.$digest();
    scope.$digest();



Консоль:


    "digest listener fired"
    "digest listener fired"
    "digest listener fired"





Ядро готово, но до конца еще далеко. Например, не учтен довольно типичный сценарий: listener-функции сами могут менять свойства из scope. Если такое случится, а за этим свойством следил другой наблюдатель, то может получиться, что этот наблюдатель не получит уведомления об изменении, по крайней мере в этот проход **$digest**:

[Код на JS Bin][10]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {}
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$digest = function() {
      var self = this;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
        }
        watch.last = newValue;
      });
    };

    var scope = new Scope();
    scope.firstName = 'Joe';
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.counter;
      },
      function(newValue, oldValue, scope) {
        scope.counterIsTwo = (newValue === 2);
      }
    );

    scope.$watch(
      function(scope) {
        return scope.firstName;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    // After the first digest the counter is 1
    scope.$digest();
    console.assert(scope.counter === 1);

    // On the next change the counter becomes two, but our other watch hasn't noticed this yet
    scope.firstName = 'Jane';
    scope.$digest();
    console.assert(scope.counter === 2);
    console.assert(scope.counterIsTwo); // false

    // Only sometime in the future, when $digest() is called again, does our other watch get run
    scope.$digest();
    console.assert(scope.counterIsTwo); // true



Консоль:


    true
    true
    false
    true





Давайте исправим это.

#### Выполняем $digest до тех пор, пока есть "грязные" данные


Нужно поправить **$digest** таким образом, чтобы он продолжал делать проверки до тех пор, пока наблюдаемые значения не перестанут меняться.

Сначала, давайте переименуем текущую функцию $digest в **$$digestOnce**, и изменим ее таким образом, чтобы она, пробегая все watch-функции один раз, возвращала булеву переменную, сообщающую было ли хоть одно изменение значений наблюдаемых полей или нет:


    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = newValue;
      });
      return dirty;
    };



После этого, заново объявим функцию **$digest**, чтобы она в цикле запускала **$$digestOnce** до тех пор пока есть изменения:


    Scope.prototype.$digest = function() {
      var dirty;
      do {
        dirty = this.$$digestOnce();
      } while (dirty);
    };



**$digest** сейчас выполняет зарегистрированные watch-функции по крайней мере один раз. Если в первом проходе, какое-либо из наблюдаемых значений изменилось, проход помечается, как «грязный», и запускается второй проход. Так происходит до тех пор, пока за весь проход не будет обнаружено ни одного измененного значения — ситуация стабилизируется.


&gt; У scope-ов в Angular, на самом деле, нет функции $$digestOnce. Вместо этого данный функционал там встрое в цикл непосредственно в $digest. Для наших целей ясность и читабельность важнее производительности, поэтому мы и сделали небольшой рефакторинг.


Вот новая реализация в действии:

[Код на JS Bin][11]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn ||function() { }
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = newValue;
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var dirty;
      do {
        dirty = this.$$digestOnce();
      } while (dirty);
    };

    var scope = new Scope();
    scope.firstName = 'Joe';
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.counter;
      },
      function(newValue, oldValue, scope) {
        scope.counterIsTwo = (newValue === 2);
      }
    );

    scope.$watch(
      function(scope) {
        return scope.firstName;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    // After the first digest the counter is 1
    scope.$digest();
    console.assert(scope.counter === 1);

    // On the next change the counter becomes two, and the other watch listener is also run because of the dirty check
    scope.firstName = 'Jane';
    scope.$digest();
    console.assert(scope.counter === 2);
    console.assert(scope.counterIsTwo);



Консоль:


    true
    true
    true





Можно сделать еще один важный вывод, касающийся watch-функций: они могут отрабатывать несколько раз в процессе работы $digest. Вот почему, часто говорят, что watch-функции должны быть идемпотентными: в функции не должно быть побочных эффектов, либо там должны быть такие побочные эффекты, для которых будет нормальным срабатывать несколько раз. Если, например, в watch-функции, есть AJAX-запрос, нет никаких гарантий на то, сколько раз этот запрос выполнится.

В нашей текущей реализации есть один большой изъян: что случитсья, если два наблюдателя будут следить за изменениями друг друга? В этом случае ситуация никогда не стабилизируется? Подобная ситуация реализована в коде ниже. В примере вызов **$digest** закомментирован.

Раскомментируйте его, чтобы узнать, что случиться:

[Код на JS Bin][12]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { }
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = newValue;
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var dirty;
      do {
        dirty = this.$$digestOnce();
      } while (dirty);
    };

    var scope = new Scope();
    scope.counter1 = 0;
    scope.counter2 = 0;

    scope.$watch(
      function(scope) {
        return scope.counter1;
      },
      function(newValue, oldValue, scope) {
        scope.counter2++;
      }
    );

    scope.$watch(
      function(scope) {
        return scope.counter2;
      },
      function(newValue, oldValue, scope) {
        scope.counter1++;
      }
    );

    // Uncomment this to run the digest
    // scope.$digest();

    console.log(scope.counter1);



Консоль:


    0





JSBin останавливает функцию через некоторое время (на моей машине происходит около 100000 итераций). Если вы запустите этот код, например, под node.js, он будет выполняться вечно.

#### Избавляемся от нестабильности в $digest


Все что нам нужно, так это ограничить работу **$digest** определенным количеством итераций. Если scope все-еще продолжает меняться после окончания итераций, мы поднимаем руки и сдаемся — вероятно состояние никогда не стабилизируется. В этой ситуации можно было бы выбросить исключение, так как состояние области видимости явно не такое, каким его ожидал видеть пользователь.

Максимальное количество итераций называется TTL (сокращение от time to live — время жизни). По умолчанию установим его равным 10. Это количество может показаться маленьким (мы только что запускали digest около 100000 раз), но учтите, это уже вопрос производительности — digest выполняется часто, и в нем каждый раз отрабатывают все watch-функции. К тому же кажется маловероятным, что у пользователя будет более 10 выстроенных в цепочку watch-функций.


&gt; В Angular TTL можно настраивать. Мы еще вернемся к этому в следующих статьях, когда будем обсуждать провайдеры и внедрение зависимостей.


Ну ладно, продолжим — давайте добавим счетчик в digest-цикл. Если достигли TTL — выбрасываем исключение:


    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };



Обновленная версия предыдущего примера выбрасывает исключение:

[Код на JS Bin][13]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { }
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (newValue !== oldValue) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = newValue;
      });
      return dirty;
    };

    Scope.prototype.$digest = function(){
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    var scope = new Scope();
    scope.counter1 = 0;
    scope.counter2 = 0;

    scope.$watch(
      function(scope) {
        return scope.counter1;
      },
      function(newValue, oldValue, scope) {
        scope.counter2++;
      }
    );

    scope.$watch(
      function(scope) {
        return scope.counter2;
      },
      function(newValue, oldValue, scope) {
        scope.counter1++;
      }
    );

    scope.$digest();



Консоль:


    "Uncaught 10 digest iterations reached (line 36)"





От зацикленности в digest избавились.

Теперь давайте посмотрим на то, _как_ именно мы определяем, что что-то поменялось.

#### Проверка на изменение по значению


На данный момент, мы сравниваем новые значения со старыми, используя оператор строгого равенства **===**. В большинстве случаев это работает: нормально определяется изменение для примитивных типов (числа, строки и т.д.), так же определяется если объект или массив заменен другим. Но в Angular есть и другой способ определения изменений, он позволяет узнать поменялось ли что-нибудь внутри массива или объекта. Для этого нужно сравнивать_ по значению_, а не _по ссылке_.

Этот тип проверки можно задействовать, передав в функцию **$watch** опциональный третий параметр булевого типа. Если этот флаг равен true — используется проверка по значению. Давайте доработаем **$watch** — будем получать флаг и сохранять его в наблюдателе (переменная **watcher**):


    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn,
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };



Все что мы сделали, это добавили наблюдателю флаг, принудительно приведя его к булеву типу, воспользовавшись двойным отрицанием. Когда пользователь вызовет **$watch** без третьего параметра, **valueEq** будет **undefined**, что преобразуется в **false** в watcher-объекте.

Проверка по значению подразумевает то, что, если значение является объектом или массивом, нужно будет пробегаться как по старому, так и по новому содержимому. Если найдутся какие-либо отличия, наблюдатель помечается, как "грязный". В содержимом могут встретиться вложенные объекты или массивы, в этом случае их тоже нужно будет рекурсивно проверять по значению.

В Angular есть своя [собственная функция сравнения по значению][14], но мы воспользуемся той, что [есть в Lo-Dash][15]. Давайте напишем функцию сравнения, принимающую пару значений и флаг:


    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue;
      }
    };



Для того чтобы определять изменения "по значению", необходимо также подругому сохранять "старые значения". Недостаточно просто хранить ссылки на текущие значения, так-как любые произведенные изменения так же попадут по ссылке и в хранимый нами объект. Мы не сможем определить поменялось что-то или нет если в функцию **$$areEqual** всегда будут попадать две ссылки на одни и те-же данные. Поэтому нам придется делать глубокое копирование содержимого, и сохранять эту копию.

Так же как и в случае с функцие сравнения, в Angular есть [своя функция глубокого копирования данных][16], но мы воспользуемся [аналогичной из Lo-Dash][17]. Давайте доработаем **$$digestOnce**, чтобы она использовала **$$areEqual** для сравнения и делала копию в **last** если нужно:


    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };



Теперь можно увидеть разницу между двумя способами сравнения значений:

[Код на JS Bin][18]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { },
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue;
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function(){
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    var scope = new Scope();
    scope.counterByRef = 0;
    scope.counterByValue = 0;
    scope.value = [1, 2, {three: [4, 5]}];

    // Set up two watches for value. One checks references, the other by value.
    scope.$watch(
      function(scope) {
        return scope.value;
      },
      function(newValue, oldValue, scope) {
        scope.counterByRef++;
      }
    );
    scope.$watch(
      function(scope) {
        return scope.value;
      },
      function(newValue, oldValue, scope) {
        scope.counterByValue++;
      },
      true
    );


    scope.$digest();
    console.assert(scope.counterByRef === 1);
    console.assert(scope.counterByValue === 1);

    // When changes are made within the value, the by-reference watcher does not notice, but the by-value watcher does.
    scope.value[2].three.push(6);
    scope.$digest();
    console.assert(scope.counterByRef === 1);
    console.assert(scope.counterByValue === 2);

    // Both watches notice when the reference changes.
    scope.value = {aNew: "value"};
    scope.$digest();
    console.assert(scope.counterByRef === 2);
    console.assert(scope.counterByValue === 3);

    delete scope.value;
    scope.$digest();
    console.assert(scope.counterByRef === 3);
    console.assert(scope.counterByValue === 4);



Консоль:


    true
    true
    true
    true
    true
    true





Проверка по значению, очевидно, более требовательна к ресурсам, чем проверка по ссылке. Перебор вложенных структур требует времени, а хранение копий объектов увеличивает расход памяти. Вот почему в Angular проверка по значению не используется по умолчанию. Вам нужно явно устанавливать флаг.


&gt; В Angular есть еще и третий механизм проверки значений на изменение: "наблюдение за коллекциями". Так же как и в механизме проверки по значениям, он выявляет изменения объектов и массивов, но в отличии от него, проверка осуществляется простая без углубления во вложенные уровни. Это естественно быстрее. Наблюдение за коллекциями доступно при помощи функции $watchCollection — ее реализацию мы рассмотрим в следующих статьях серии.


Прежде чем мы закончим со сравнением значений необходимо учесть одну особенность JavaScript.

#### Значения NaN


В языке JavaScript значение **NaN** (not a number — не число) не равно самому себе. Это может звучать странно, наверное, потому что так оно и есть. Мы не обрабатывали **NaN** вручную в нашей функции проверки значений на изменения, поэтому watch-функция, наблюдающая за **NaN** всегда будет помечать наблюдатель, как "грязный".

В проверке "по значению" этот случай уже учтен в функции **isEqual** из Lo-Dash. В проверке "по ссылке" нам придется сделать это самим. Что же давайте доработаем функцию **$$areEqual**:


    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };



Теперь наблюдатели с **NaN** ведут себя как положено:

[Код на JS Bin][19]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function(){
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    var scope = new Scope();
    scope.number = 0;
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.number;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );


    scope.$digest();
    console.assert(scope.counter === 1);

    scope.number = parseInt('wat', 10); // Becomes NaN
    scope.$digest();
    console.assert(scope.counter === 2);



Консоль:


    true
    true





Теперь давайте сместим фокус с проверки значений на то, каким образом можно взаимодействовать со scope из кода приложений.

#### $eval — выполнение кода в контексте scope


В Angular есть несколько вариантов запуска кода в контексте scope. Простейший из них — это функция **$eval**. Она принимает функцию в качестве аргумента, и единственное, что делает — это сразу же ее вызывает, передавая ей текущий scope, как параметр. Ну а потом она возвращает результат выполнения. **$eval** также принимает второй параметр, который она без изменений передает в вызываемую функцию.

Реализация **$eval** очень простая:


    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };


Использование **$eval** так же довольно просто:

[Код на JS Bin][20]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function(){
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    var scope = new Scope();
    scope.number = 1;

    scope.$eval(function(theScope) {
      console.log('Number during $eval:', theScope.number);
    });



Консоль:


    "Number during $eval:"
    1





Так в чем же польза от такого вычурного способа вызова функции? Одно из преимуществ состоит в том, что **$eval** делает чуть более прозрачным код, работающий с содержимым scope. Так же **$eval** является составным блоком для **$apply**, которым мы вскоре займемся.

Однако, самая большая польза от **$eval** проявится только когда мы начнем обсуждать использование "выражений" вместо функций. Так же как и в случае с **$watch**, в функцию **$eval** можно передавать строковое выражение. Она его скомпилирует и выполнит в контексте scope. Дальше в серии статей, мы реализуем это.

#### $apply — интеграция внешнего кода с циклом $digest


Вероятно, **$apply** самая известная из всех функций **Scope**. Она позиционируется, как стандартный способ интеграции сторонних библиотек c Angular. И для этого есть причины.

$apply принимает функцию как аргумент, вызывает эту функцию, используя **$eval**, ну а в конце запускает **$digest**. Вот ее простейшая реализация:


    Scope.prototype.$apply = function(expr) {
      try {
        return this.$eval(expr);
      } finally {
        this.$digest();
      }
    };



**$digest** вызывается в блоке **finally** для того, чтобы обновить зависимости, даже если в функции произошли исключения.

Идея состоит в том, что используя **$apply**, мы можем выполнять код, не знакомый с Angular. Этот код может менять данные в scope, а **$apply** позаботится о том, чтобы наблюдатели подхватили эти изменения. Именно эти и имеют ввиду, когда говорят об «интеграции кода в жизненный цикл Angular». Это и ничего больше.

**$apply** в действии:

[Код на JS Bin][21]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function(){
      var ttl = 10;
      var dirty;
      do {
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        return this.$eval(expr);
      } finally {
        this.$digest();
      }
    };

    var scope = new Scope();
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.aValue;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    scope.$apply(function(scope) {
      scope.aValue = 'Hello from "outside"';
    });
    console.assert(scope.counter === 1);



Консоль:


    true




#### Отложенное выполнение — $evalAsync


В JavaScript часто бывает необходимо выполнить участок кода «позже» — то есть отложить выполнение до того момента, когда весь код текущего контекста выполнения будет выполнен. Обычно это делают используя **SetTimeout()** с нулевой (или близкой к нулю) задержкой.

Данный прием работает и в Angular приложениях, хотя и предпочтительнее для этого использовать [сервис][22] **$timeout**, который кроме всего прочего интегрирует вызов отложенной функции с digest-циклом при помощи **$apply**.

Но есть еще один способ отложенного выполнения кода в Angular — это функция **$evalAsync**. Она принимает в качестве параметра функцию, и обеспечивает ее выполнение позже, но либо прям внутри текущего цикла digest (если он сейчас отрабатывает), либо же непосредственно перед следующим digest-циклом. Вы можете, например, отложить выполнение какого-либо кода непосредственно из listener-функции наблюдателя, зная, что, несмотря на то, что код отложен, он будет выполнен на следующей итерации digest-цикла.

Прежде всего нужно определиться с тем, где мы будем хранить задачи, отложенные через **$$evalAsync**. Можно использовать для этого массив, инициализировав его в конструкторе **Scope**:


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
    }



Далее напишем саму **$evalAsync**, которая будет добавлять функции в очередь:


    Scope.prototype.$evalAsync = function(expr) {
      this.$$asyncQueue.push({scope: this, expression: expr});
    };





&gt; Причина по которой мы явным образом добавляем scope в объект очереди связана с наследованием областей видимости (scope-ов), которое мы будем обсуждать в следующей статье данной серии.


Теперь первое, что мы будем делать в **$digest**, это извлекать все функции, которые есть в очереди отложенного запуска, и выполнять их, используя **$eval**:


    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };



Эта реализация гарантирует, что если вы отложили выполнение функции, а scope был помечен, как "грязный" — функция будет вызвана отложено, но в том же digest-цикле.

Вот пример того, как **$evalAsync** может использоваться:

[Код на JS Bin][23]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
    }

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          throw "10 digest iterations reached";
        }
      } while (dirty);
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        return this.$eval(expr);
      } finally {
        this.$digest();
      }
    };

    Scope.prototype.$evalAsync = function(expr) {
      this.$$asyncQueue.push({scope: this, expression: expr});
    };

    var scope = new Scope();
    scope.asyncEvaled = false;

    scope.$watch(
      function(scope) {
        return scope.aValue;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
        scope.$evalAsync(function(scope) {
          scope.asyncEvaled = true;
        });
        console.log("Evaled inside listener: "+scope.asyncEvaled);
      }
    );

    scope.aValue = "test";
    scope.$digest();
    console.log("Evaled after digest: "+scope.asyncEvaled);



Консоль:


    "Evaled inside listener: false"
    "Evaled after digest: true"







#### Фазы в scope


Функция **$evalAsync** делает еще кое-что, она должна запланировать выполнение digest, если он сейчас не выполняется. Смысл этого в том, что когда бы вы не вызвали **$evalAsync**, вы должны быть уверены, что ваша отложенная функция выполнится «довольно скоро», а не тогда, когда что-нибудь еще запустит digest.

**$evalAsync** должна как-то понимать, запущен сейчас digest или нет. Для этой цели в Angular scope реализован механизм, называемый «фаза», который представляет собой обычную строку в scope, в которой хранится информация о том, что сейчас происходит.

Внесем в конструктор **$scope** поле **$$phase**, установив его в **null**:


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$phase = null;
    }



Далее давайте напишем пару функций для контроля фазы: одну для установки, другую для очистки. Также добавим дополнительную проверку на то, что никто не пытается установить фазу, не закончив предыдущую:


    Scope.prototype.$beginPhase = function(phase) {
      if (this.$$phase) {
        throw this.$$phase + ' already in progress.';
      }
      this.$$phase = phase;
    };

    Scope.prototype.$clearPhase = function() {
      this.$$phase = null;
    };



В функции **$digest** установим фазу "$digest", обернем в нее digest-цикл:


    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();
    };



Пока мы здесь, давайте заодно доработаем **$apply**, чтобы и тут прописывалась фаза. Это будет полезно в процессе отладки:


    Scope.prototype.$apply = function(expr) {
      try {
        this.$beginPhase("$apply");
        return this.$eval(expr);
      } finally {
        this.$clearPhase();
        this.$digest();
      }
    };



Теперь наконец можно запланировать вызов **$digest** в функции **$evalAsync**. Здесь нужно будет проверить фазу, если она пуста (и ни одной асинхронной задачи еще не запланировано) — планируем выполнение **$digest**:


    Scope.prototype.$evalAsync = function(expr) {
      var self = this;
      if (!self.$$phase &amp;&amp; !self.$$asyncQueue.length) {
        setTimeout(function() {
          if (self.$$asyncQueue.length) {
            self.$digest();
          }
        }, 0);
      }
      self.$$asyncQueue.push({scope: self, expression: expr});
    };



В этой реализации, вызывая **$evalAsync**, можно быть уверенным в том, что digest произойдет в ближайшее время, вне зависимости от того, откуда произошел вызов:

[Код на JS Bin][24]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$phase = null;
    }

    Scope.prototype.$beginPhase = function(phase) {
      if (this.$$phase) {
        throw this.$$phase + ' already in progress.';
      }
      this.$$phase = phase;
    };

    Scope.prototype.$clearPhase = function() {
      this.$$phase = null;
    };

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        this.$beginPhase("$apply");
        return this.$eval(expr);
      } finally {
        this.$clearPhase();
        this.$digest();
      }
    };

    Scope.prototype.$evalAsync = function(expr) {
      var self = this;
      if (!self.$$phase &amp;&amp; !self.$$asyncQueue.length) {
        setTimeout(function() {
          if (self.$$asyncQueue.length) {
            self.$digest();
          }
        }, 0);
      }
      self.$$asyncQueue.push({scope: self, expression: expr});
    };

    var scope = new Scope();
    scope.asyncEvaled = false;

    scope.$evalAsync(function(scope) {
      scope.asyncEvaled = true;
    });

    setTimeout(function() {
      console.log("Evaled after a while: "+scope.asyncEvaled);
    }, 100); // Check after a delay to make sure the digest has had a chance to run.



Консоль:


    "Evaled after a while: true"







#### Запуск кода после digest — $$postDigest


Есть еще один способ добавить свой код в поток выполнения digest-цикла — используя функцию **$$postDigest**.

Двойной доллар в начале имени функции говорит о том, что это глубинная Angular-функция, которую не должны использовать разработчики Angular-приложения. Но нам это не важно, мы все равно ее реализуем.

Так же как и **$evalAsync**, **$$postDigest** позволяет отложить запуск какого-то кода на "потом". Более конкретно, отложенная функция будет выполнена сразу после того, как следующий digest будет завершен. Использование **$$postDigest** не подразумевает принудительного запуска **$digest**, поэтому запуск отложенной функции может задержаться до того момента, когда какой-нибудь сторонний код не инициирует digest. Как имя и подразумевает, **$$postDigest** всего-лишь запускает отложенные функции сразу после digest, поэтому если вы модифицировали scope в коде, передаваемом в **$$postDigest**, вам нужно явно использовать **$digest** или **$apply**, чтобы изменения подхватились.

Для начала, давайте добавим еще одну очередь в конструктор **Scope**, на этот раз для **$$postDigest**:


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$postDigestQueue = [];
      this.$$phase = null;
    }



Далее, реализуем саму **$$postDigest**. Все что она делает, это добавляет принимаемую функцию в очередь:


    Scope.prototype.$$postDigest = function(fn) {
      this.$$postDigestQueue.push(fn);
    };



Ну и в завершение, в конце **$digest**, мы должны вызвать все функции за раз и очистить очередь:


    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();

      while (this.$$postDigestQueue.length) {
        this.$$postDigestQueue.shift()();
      }
    };



Вот пример, как можно использовать функцию **$$postDigest**:

[Код на JS Bin][25]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$postDigestQueue = [];
      this.$$phase = null;
    }

    Scope.prototype.$beginPhase = function(phase) {
      if (this.$$phase) {
        throw this.$$phase + ' already in progress.';
      }
      this.$$phase = phase;
    };

    Scope.prototype.$clearPhase = function() {
      this.$$phase = null;
    };

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        var newValue = watch.watchFn(self);
        var oldValue = watch.last;
        if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
          watch.listenerFn(newValue, oldValue, self);
          dirty = true;
        }
        watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          var asyncTask = this.$$asyncQueue.shift();
          this.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();

      while (this.$$postDigestQueue.length) {
        this.$$postDigestQueue.shift()();
      }
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        this.$beginPhase("$apply");
        return this.$eval(expr);
      } finally {
        this.$clearPhase();
        this.$digest();
      }
    };

    Scope.prototype.$evalAsync = function(expr) {
      var self = this;
      if (!self.$$phase &amp;&amp; !self.$$asyncQueue.length) {
        setTimeout(function() {
          if (self.$$asyncQueue.length) {
            self.$digest();
          }
        }, 0);
      }
      self.$$asyncQueue.push({scope: self, expression: expr});
    };

    Scope.prototype.$$postDigest = function(fn) {
      this.$$postDigestQueue.push(fn);
    };


    var scope = new Scope();
    var postDigestInvoked = false;

    scope.$$postDigest(function() {
      postDigestInvoked = true;
    });

    console.assert(!postDigestInvoked);

    scope.$digest();
    console.assert(postDigestInvoked);



Консоль:


    true
    true







#### Обработка исключений


Наша текущая реализация **$scope** все больше и больше приближается к версии в Angular. Однако она еще довольно хрупка. Это оттого, что мы не уделяли достаточно внимания обработке исключений.

Scope-объекты в Angular довольно устойчивы к ошибкам: когда возникают исключения в watch-функциях, **$evalAsync** или в **$$postDigest** — это не прерывает digest-цикл. В нашей текущей реализации любая из эти ошибок выбросит нас из digest.

Можно достаточно легко исправить это, обернув изнутри вызывающий блок всех этих функции в **try…catch**


&gt; В Angular эти ошибки передаются в специальный сервис $exceptionHandler. У нас его пока нет, так что мы пока просто будем выводить ошибки в консоль.


Обработка исключений для **$evalAsync** и **$$postDigest** делается в функции **$digest**. В обоих случаях исключение логируется, а digest продолжается нормально:


    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          try {
            var asyncTask = this.$$asyncQueue.shift();
            this.$eval(asyncTask.expression);
          } catch (e) {
            (console.error || console.log)(e);
          }
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();

      while (this.$$postDigestQueue.length) {
        try {
          this.$$postDigestQueue.shift()();
        } catch (e) {
          (console.error || console.log)(e);
        }
      }
    };



Обработка исключений для watch-функция делается в **$digestOnce**:


    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        try {
          var newValue = watch.watchFn(self);
          var oldValue = watch.last;
          if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
            watch.listenerFn(newValue, oldValue, self);
            dirty = true;
          }
          watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
        } catch (e) {
          (console.error || console.log)(e);
        }
      });
      return dirty;
    };




Теперь наш digest-цикл намного надежней к исключениям:

[Код на JS Bin][26]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$postDigestQueue = [];
      this.$$phase = null;
    }

    Scope.prototype.$beginPhase = function(phase) {
      if (this.$$phase) {
        throw this.$$phase + ' already in progress.';
      }
      this.$$phase = phase;
    };

    Scope.prototype.$clearPhase = function() {
      this.$$phase = null;
    };

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() {},
        valueEq: !!valueEq
      };
      this.$$watchers.push(watcher);
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        try {
          var newValue = watch.watchFn(self);
          var oldValue = watch.last;
          if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
            watch.listenerFn(newValue, oldValue, self);
            dirty = true;
          }
          watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
        } catch (e) {
          (console.error || console.log)(e);
        }
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          try {
            var asyncTask = this.$$asyncQueue.shift();
            this.$eval(asyncTask.expression);
          } catch (e) {
            (console.error || console.log)(e);
          }
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();

      while (this.$$postDigestQueue.length) {
        try {
          this.$$postDigestQueue.shift()();
        } catch (e) {
          (console.error || console.log)(e);
        }
      }
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        this.$beginPhase("$apply");
        return this.$eval(expr);
      } finally {
        this.$clearPhase();
        this.$digest();
      }
    };

    Scope.prototype.$evalAsync = function(expr) {
      var self = this;
      if (!self.$$phase &amp;&amp; !self.$$asyncQueue.length) {
        setTimeout(function() {
          if (self.$$asyncQueue.length) {
            self.$digest();
          }
        }, 0);
      }
      self.$$asyncQueue.push({scope: self, expression: expr});
    };

    Scope.prototype.$$postDigest = function(fn) {
      this.$$postDigestQueue.push(fn);
    };


    var scope = new Scope();
    scope.aValue = "abc";
    scope.counter = 0;

    scope.$watch(function() {
      throw "Watch fail";
    });
    scope.$watch(
      function(scope) {
        scope.$evalAsync(function(scope) {
          throw "async fail";
        });
        return scope.aValue;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    scope.$digest();
    console.assert(scope.counter === 1);



Консоль:


    "Watch fail"
    "async fail"
    "Watch fail"
    true







#### Отключение наблюдателя


Регистрируя наблюдатель, в большинстве случаев, вам нужно, чтобы он оставалась активным все время жизни scope-объекта, и нет необходимости явным образом удалять его. Но в некоторых случаях может потребоваться удалить какой-либо наблюдатель, в то время, как scope должен продолжать работать.

Функция **$watch** в Angular на самом деле возвращает значение — функцию, вызов которой, удаляет зарегистрированный наблюдатель. Чтобы реализовать это, все что нам нужно, это чтобы **$watch** возвращала функцию, удаляющую только что созданный наблюдатель из массива **$$watchers**:


    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var self = this;
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn,
        valueEq: !!valueEq
      };
      self.$$watchers.push(watcher);
      return function() {
        var index = self.$$watchers.indexOf(watcher);
        if (index &gt;= 0) {
          self.$$watchers.splice(index, 1);
        }
      };
    };




Теперь можно запомнить возвращаемую из **$watch** функцию, и вызвать ее позже, когда нужно будет уничтожить наблюдатель:

[Код на JS Bin][27]


**Просмотреть код**


    function Scope() {
      this.$$watchers = [];
      this.$$asyncQueue = [];
      this.$$postDigestQueue = [];
      this.$$phase = null;
    }

    Scope.prototype.$beginPhase = function(phase) {
      if (this.$$phase) {
        throw this.$$phase + ' already in progress.';
      }
      this.$$phase = phase;
    };

    Scope.prototype.$clearPhase = function() {
      this.$$phase = null;
    };

    Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
      var self = this;
      var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { },
        valueEq: !!valueEq
      };
      self.$$watchers.push(watcher);
      return function() {
        var index = self.$$watchers.indexOf(watcher);
        if (index &gt;= 0) {
          self.$$watchers.splice(index, 1);
        }
      };
    };

    Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
      if (valueEq) {
        return _.isEqual(newValue, oldValue);
      } else {
        return newValue === oldValue ||
          (typeof newValue === 'number' &amp;&amp; typeof oldValue === 'number' &amp;&amp;
           isNaN(newValue) &amp;&amp; isNaN(oldValue));
      }
    };

    Scope.prototype.$$digestOnce = function() {
      var self  = this;
      var dirty;
      _.forEach(this.$$watchers, function(watch) {
        try {
          var newValue = watch.watchFn(self);
          var oldValue = watch.last;
          if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
            watch.listenerFn(newValue, oldValue, self);
            dirty = true;
          }
          watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
        } catch (e) {
          (console.error || console.log)(e);
        }
      });
      return dirty;
    };

    Scope.prototype.$digest = function() {
      var ttl = 10;
      var dirty;
      this.$beginPhase("$digest");
      do {
        while (this.$$asyncQueue.length) {
          try {
            var asyncTask = this.$$asyncQueue.shift();
            this.$eval(asyncTask.expression);
          } catch (e) {
            (console.error || console.log)(e);
          }
        }
        dirty = this.$$digestOnce();
        if (dirty &amp;&amp; !(ttl--)) {
          this.$clearPhase();
          throw "10 digest iterations reached";
        }
      } while (dirty);
      this.$clearPhase();

      while (this.$$postDigestQueue.length) {
        try {
          this.$$postDigestQueue.shift()();
        } catch (e) {
          (console.error || console.log)(e);
        }
      }
    };

    Scope.prototype.$eval = function(expr, locals) {
      return expr(this, locals);
    };

    Scope.prototype.$apply = function(expr) {
      try {
        this.$beginPhase("$apply");
        return this.$eval(expr);
      } finally {
        this.$clearPhase();
        this.$digest();
      }
    };

    Scope.prototype.$evalAsync = function(expr) {
      var self = this;
      if (!self.$$phase &amp;&amp; !self.$$asyncQueue.length) {
        setTimeout(function() {
          if (self.$$asyncQueue.length) {
            self.$digest();
          }
        }, 0);
      }
      self.$$asyncQueue.push({scope: self, expression: expr});
    };

    Scope.prototype.$$postDigest = function(fn) {
      this.$$postDigestQueue.push(fn);
    };


    var scope = new Scope();
    scope.aValue = "abc";
    scope.counter = 0;

    var removeWatch = scope.$watch(
      function(scope) {
        return scope.aValue;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    scope.$digest();
    console.assert(scope.counter === 1);

    scope.aValue = 'def';
    scope.$digest();
    console.assert(scope.counter === 2);

    removeWatch();
    scope.aValue = 'ghi';
    scope.$digest();
    console.assert(scope.counter === 2); // No longer incrementing



Консоль:


    true
    true
    true







#### Что дальше


Мы проделали долгий путь, и создали отличную реализацию scope-объектов, в лучших традициях Angular. Но в scope-объекты в Angular — намного больше, чем то, что есть у нас.

Наверное важнее всего то, что scope в Angular, это не обособленные независимые объекты. Наоборот, scope-объекты наследуют от других scope-ов, а наблюдатели могут следить не только за свойствами из scope, к которому они привязаны, но и за свойствами родительских scope-ов. Этот подход, такой простой по сути — источник многих проблем у начинающих. Именно поэтому наследование областей видимости (scope) станет предметом исследования следующей статьи данной серии.

В дальнейшем мы также обсудим подсистему событий, которая тоже реализована в **Scope**.

**От переводчика:**

Текст довольно большой, думаю есть и ошибки и опечатки. Шлите их в личку — все поправлю.

Если кого-то знает, как на хабре выделять строки в коде — говорите, это улучшит читабельность кода.

[1]: http://docs.angularjs.org/
[2]: http://syntaxspectrum.com/tag/angularjs/
[3]: https://github.com/teropa/schmangular.js
[4]: http://jsbin.com/
[5]: http://lodash.com/
[6]: http://jsbin.com/UGOVUk/4/edit
[7]: http://jsbin.com/oMaQoxa/2/edit
[8]: http://jsbin.com/OsITIZu/3/edit
[9]: http://jsbin.com/OsITIZu/4/edit
[10]: http://jsbin.com/eTIpUyE/2/edit
[11]: http://jsbin.com/Imoyosa/3/edit
[12]: http://jsbin.com/eKEvOYa/3/edit
[13]: http://jsbin.com/uNapUWe/2/edit
[14]: https://github.com/angular/angular.js/blob/8d4e3fdd31eabadd87db38aa0590253e14791956/src/Angular.js#L812
[15]: http://lodash.com/docs#isEqual
[16]: https://github.com/angular/angular.js/blob/8d4e3fdd31eabadd87db38aa0590253e14791956/src/Angular.js#L725
[17]: http://lodash.com/docs#cloneDeep
[18]: http://jsbin.com/ARiWENO/3/edit
[19]: http://jsbin.com/ijINaRA/2/edit
[20]: http://jsbin.com/UzaWUC/1/edit
[21]: http://jsbin.com/UzaWUC/2/edit
[22]: http://docs.angularjs.org/api/ng.$timeout
[23]: http://jsbin.com/ilepOwI/1/edit
[24]: http://jsbin.com/iKeSaGi/1/edit
[25]: http://jsbin.com/IMEhowO/1/edit
[26]: http://jsbin.com/IMEhowO/2/edit
[27]: http://jsbin.com/IMEhowO/4/edit
  