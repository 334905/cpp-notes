# Наследование и виртуальные функции
- [Запись лекции №1](https://www.youtube.com/watch?v=IcAaaX888xc)
- [Запись лекции №2](https://www.youtube.com/watch?v=11MKhMYAmnE)
- [Запись лекции №3](https://www.youtube.com/watch?v=oMkF60mU8ig)
- [Запись лекции №4](https://www.youtube.com/watch?v=0-92_jC7YMU)
---
```c++
struct vehicle {
	std::string registration_number;
	void print_name() {
		std::cout << "vehicle\n";
	}
};
struct bus : vehicle {	// наследуется от vehicle
	int32_t route_number;
	std::string next_stop() const;
	void print_name() {
		std::cout << "bus\n";
	}
};
struct truck : vehicle {
	double cargo_mass;
	void print_name() {
		std::cout << "truck\n";
	}
};
void f(vehicle& v){
	v.print_name(); // выведет vehicle, так как статический тип vehicle и при компиляции f подставляется вызов print_name от базового класса
}
int main() {
	bus b;
	vehicle& v = b; // можно делать так
	v.registation_number = "123";
}
```

`bus` содержит и `route_number`, и `registration_number`.
Компилятор ищет методы и поля сначала внутри последнего, наследовавшего класса. Поэтому обращение к одинаковым полям класса будет возвращать поле последнего. Если наследуемся от двух классов (в C++ есть множественное наследование) и в двух базовых есть поле/метод с одинаковым названием, а в наследнике нет, то будет ошибка компиляции при обращении от объекта наследника.\
Если поля классов совпадают в названии, то обратиться к полю другого можно либо через приведение, либо через `classname::field`.\
Наследующий класс можно приводить к любому наследуемому классу и наоборот.

С методами в наследовании работают точно так же, как и с полями:

```c++
struct base {
	void g();
};
struct derived : base {
	void g();
};
int main() {
	derived d;
	d.g(); // запустит g() из derived
	d.base::g(); // запустит g() из base
}
```
При создании объектов класса-наследника вызывается дефолтный конструктор базового класса (классы конструируются в порядке от базового к производному, деструкторы наоборот). Если хотим вызывать какой-то определённый конструктор, то можно вызывать его через списки инициализации:

```c++
struct bus : vehicle {
	bus(std::string const& registration_number, std::string const& next_stop_name) 
		: vehicle(registration_number),
			next_stop_name(next_stop_name){}
}
```
При этом стандарт определяет следующий порядок инициализации [перевод с cppreference](https://en.cppreference.com/w/cpp/language/constructor):
- Инициализируются виртуальные базовые классы (про них будет потом) in depth-first left-to-right traversal.
- Инициализируются прямые базовые классы in left-to-right order.
- Инициализируются нестатичные члены класса в порядке их объявления.

**Как не нужно наследоваться**: если вы не добавляете новых данных, то, возможно, вам не нужно наследование и хватит просто функций.


## Виртуальные функции

Когда без наследования не обойтись? Чаще всего его используют вместе с такими конструкциями, как виртуальные функции. Реализованы они через так называемую таблицу виртуальных функций.

Функции можно пометить кодовым словом `virtual`. Если пометить в базовом классе, то в производных они тоже будут считаться `virtual`. Функция `virtual` вызывается в соответствии с динамическим, а не статическим типом (определяется в рантайме).

```c++
struct vehicle {
	virtual void print_name() {
		std::cout << "vehicle" << std::endl;
	}
}
struct bus : vehicle {
	void print_name() {
  	std::cout << "bus" << std::endl;
  }
}
void f (vehicle& a) {
	a.print_name();
}
int main() {
	bus b;
  b.print_name();
  f(b); // выведет bus, так как print_name виртуальная
}
```
Чаще всего для базового класса тяжело разумно определить копирование и присваивание, поэтому помечаем их `delete`:
```c++
int main() {
	bus b;
	vehicle v = b; // что тут происходит? это аналоогично vehicle v = (vehicle&) b; и вызывается конструктор копирования у vehicle
}
```

Деструктор базового класса лучше объявлять виртуальным: в таком случае обеспечивается правильное разрушение объектов (чтобы всегда вызывался конструктор производного класса).

**Мем**: параметры по умолчанию являются частью декларации, поэтому соответствуют статическому типу, даже если указать другие в наследнике.

```c++
#include <string>
#include <iostream>
struct vehicle {
	virtual void print_name(std::string prefix = "Base: "){
		std::cout << prefix << "vehicle" << std::endl;
	}
};
struct bus : vehicle {
	void print_name(std::string prefix = "Derived: "){
		std::cout << prefix << "bus" << std::endl;
	}
};
void bar(vehicle& t){
	t.print_name();
}
int main() {
	bus b;
	b.print_name(); // Derived: bus
	bar(b);	// Base: bus
}
```
### Про то, как это устроено внутри:
Мы могли бы вместо виртуальных функций делать указатели на функции, но это дорого, так как каждая функция добавляет указатель каждому объекту. Но так как для каждого типа они у нас общие, они хранятся в таблице виртуальных функций, а в объекте хранится указатель на неё.

Из-за этого наличие хотя бы одной виртуальной функции в классе добавляет указатель объектам, а при вызове функции сначала смотрим в таблицу, что обходится дороже и мешает компилятору инлайнить функции.

## Абстрактные методы (и классы):

Если в классе есть хотя бы одна абстрактная функция (ещё их называют чистыми виртуальными), то класс считается абстрактным и создавать объекты такого типа нельзя.

```c++
struct output_device {
	virtual void write(void const* data, size_t size) = 0; // обязательство реализации у наследников
};
struct speakers : output_device {};
struct twitch_stream : output_device {};
struct null_output : output_device {};
```

## Ещё немного про удаление:

```c++
struct base1 {
	int a;
};
struct base2 {
	int b;
};
struct derived : base1, base2 {};
int main () {
	derived* d = new derived();
	base2* b = d;
  delete b; // UB, скорее всего получим ошибку о том, что b - невалидный указатель
}
```

Почему ошибка именно такая? Это из-за того, как наследование реализовано внутри: по указателю на `derived` подряд лежат `base1` и `base2` (можно понять, если посмотреть, во что компилируется приведение к ним). По сути, в `delete b` мы пытаемся освободить не то, что аллоцировали, а то, что лежит по указателю `derived+4`. 

Ещё можно заметить, что приведение ссылок и указателей компилируются по-разному, потому что `nullptr` после приведения должен оставаться `nullptr`.

## Приведение типов (cast):

Представим, что у нас получился такой код (такое иногда бывает, если части приходят из разных хедеров):

```c++
struct base1 {
	int a;
};
struct base2 {
	int b;
};
struct derived;
derived& to_derived(base2& b) {
	return (derived&)b;
}
struct derived : base1, base2 {};
derived& to_derived_2(base2& b) {
	return (derived&)b;
}
```

Функции `to_derived` и `to_derived_2` скомпилируются в разный код, потому что на момент компиляции `to_derived` типы `base2` и `derived` - несвязанные и Си-шные касты втупую приводят указатели для них без пересчёта сдвигов в памяти.

Как такого избежать (хотя бы словить ошибку)? Использовать C++-style касты:

```c++
derived& to_derived(base2& b) {
	return static_cast<derived&>(b); // синтаксис шаблонов
}
```

Если это будет написано в том же месте, то словим ошибку компиляции, если после определения `derived`, то проблем не будет.

Всего есть 4 разных `cast`:

- `static_cast` - то, что нам нужно в 99% случаев. В основном это те касты, которые адекватно прописаны в стандарте: касты чисел, upcast/downcast (в наследовании), T* <-> void*.

- `reinterpret_cast` - касты, которые зависят от реализации компилятором. Редко бывает полезен, стоит избегать. Например, может приводить указатели к числам.

- `const_cast` - тоже редко хотим использовать. Например:

```c++
int a;
int const& b = a;	// знаем, что ссылка на неконстантный a, но по ней нельзя менять
int& c = const_cast<int&>(b); // "снимает const", редко нужно, например, если в библиотеке забыли пометить аргумент функции const, а хотим передать что-то константно
```

- `dynamic_cast` - позволяет кастить к указателям и ссылкам на объекты полиморфного типа (которые содержат хотя бы одну виртуальную функцию). Если динамический тип не приводится к тому, к чему кастуем, то `nullptr` (для ссылок бросит исключение).

```c++
struct base {
	virtual ~base();
};
struct derived : base {};
void f(base* b) {
	derived* d = dynamic_cast<derived*>(b);
}
```

Как это работает внутри? Если посмотреть на код, в который это компилируется, в качестве параметров `dynamic_cast` передаются `type_info` для классов. `type_info` содержит информацию об имени класса и позволяет проверить, одинаковый ли тип у объектов, получить его можнно через `typeid(d)`. Указатель на `type_info` хранится в таблице виртуальных функций.

*В коде редко используют `dynamic_cast`, потому что чаще всего можно обойтись без него и ещё это может быть дорого*\
Ключ компиляции `-fno-rtti` отключает хранение рантайм-информации, если хочется сэкономить место в бинарном файле, но это выключает возможность использовать 

## Private наследование

```c++
struct output_device {
	virtual void write(void const* data, size_t size) = 0;
	virtual void set_volume(double val) = 0;
	virtual void write(void const* data, size_t size) = 0;
};
struct volume_data : output_device {
	void set_volume(double val) override {
		volume = val;
	}
	double get_volume() override {
		return volume;
	}
private:
	double volume;
}
struct file : volume_data {};
struct speakers : volume_data {};
```

В коде выше можно приводить `file` и `speakers` к `volume_data` или использовать указатели на них как указатели на `volume_data`, чего мы не хотим. Эту проблему решает `private` наследование:

```c++
struct file : private volume_data {}; 
```

`private` base класс означает, что мы знаем про наследование только внутри класса. Тогда снаружи нельзя не только приводить, но и вызывать функции родительского класса (если нет `override` в наследнике).

В этом примере есть проблема - от `output_device` в таком случае мы тоже наследуемся приватно, так как `private` "скрывает" непрямые базовые классы тоже. 

Часто для использования как выше, можно делать так (например, если у нас независимые аспекты поведения, *ну ещё можно менять в рантайме реализацию*):

```c++
struct widget_painter {
	virtual void paint() = 0;
};
struct widget {
	widget(widget_painter*);
	void set_painter(widget_painter*);
private:
	widget_painter* painter;	// аспекты поведения меняем, передавая разные widget_painter
};
struct mandelbrot_painter : widget_painter {
	void print() override;
};
```

### Protected

`Protected` - модификатор, как `public` и `private`, но `protected` функции можно вызывать только в классах-наследниках.

### Мем про квадрат и прямоугольник

Как правильно наследоваться: квадрат от прямоугольника или прямоугольника от квадрата? 

~~А никак~~ Это зависит от того, что требуется от интерфейса. В общем случае и то, и то может не подходить (у `rectangle` разные стороны, у `square` одинаковые, поэтому могут быть проблемы с сеттерами/геттерами и другими функциями)

## Virtual наследование

Если базовый класс помечен `virtual`, то это значит, что он шарится с другими классами в иерархии. Для иерархии все `virtual` базы склеиваются в один `subobject`. Пример:

```c++
struct a {
	int x;
	void f() {
		x = 42;
	}
};
struct b : virtual a {};
struct c : virtual a {};
struct d : b, c {};
int main() {
     d dd;
     dd.f(); // не упадёт, как без virtual, так как a общий
}
```

Но нельзя в `b` и `c` оверрайдить одинаковые виртуальные функции из `a`, будет ошибка компиляции.

С помощью `virtual` наследования можно реализовывать часть функций интерфейса:

```c++
struct a {
	virtual void f() = 0;
	virtual void g() = 0;
	virtual void h() = 0;
};
struct f_impl : virtual a {
	void f() override;
};
struct g_impl : virtual a {
	void g() override;
};
struct derived : f_impl, g_impl {
	void h() override;
};
```

Как хранится виртуальная база? Хранятся указатели на таблицу, в которой оффсеты каждой виртуальной базы.