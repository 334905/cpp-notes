# Классы, абстракция данных
- [Пример к лекции](https://github.com/sorokin/cpp-course/blob/gh-pages/demos/string-demo/main.cpp)
- [Запись лекции №1](https://www.youtube.com/watch?v=4LkTiNYQYBU)
- [Запись лекции №2](https://www.youtube.com/watch?v=kjJ-1-VsNRo)
---
- [Пример к лекции](https://github.com/sorokin/cpp-course/blob/gh-pages/demos/string-demo/main.cpp)
- [Запись лекции №3](https://www.youtube.com/watch?v=nI6NEPYPRXU)
- [Запись лекции №4](https://www.youtube.com/watch?v=8JAp3tG6IrA)
---

## Методы.
Как бы мы с нашими текущими знаниями реализовали структуру для комплексных чисел. Ну, как-то так:
```c++
struct complex {
	double re;
	double im;
};
void conjugate(complex* c) {
	c->im = -c->im;
}
```
Это корректный и хороший C'шный код. Но в C++ позволили писать функции внутри класса. Они, как в Java принимают неявный параметр `this`, который указатель на «себя». При этом `this->` можно опустить везде, где он есть. То есть в C++ код выше был бы написан так:
```c++
struct complex {
	double re;
	double im;

	void conjugate() {
		im = -im;     // this->im = -this->im;
	}
};
```
Компилятор генерирует один и тот же код для программы в C'шном стиле и для программы в C++ стиле.

Также важный момент: **когда вы хотите указать, что `this` имеет тип `const complex* const` (а не `complex* const`), вы пишете `const` после закрывающей скобки аргументов функции**.

## Немного про компиляцию классов.
Кстати. Можно написать сначала метод, а потом поля, это ни на что не влияет. Почему? А потому что компилятор откладывает разбор тел функций на конец класса. Но тут же возникает вопрос, почему так не делали с обычными функциями?

1. По историческим причинам. Когда у компьютеров было мало памяти, такие штуки компиляторы вообще никак не могли себе позволить. Поэтому по развитию GCC можно посмотреть,
что сначала он оптимизировал только одну функцию за раз, потом только одну единицу трансляции, а теперь уже у нас есть LTO. Понятно, что если с LTO компилировать что-то огромное, вам нужно будет гигов этак двадцать.
2. Второй аргумент — хочется прекомпилировать заголовки. *Прекомпиляция заголовков* — это когда вы проходитесь по заголовку один раз, сохраняете состояние компилятора и потом загружаете его, чтобы не разбирать заголовок снова. И если бы заголовки зависели от того, что после них, такое было бы невозможно.

Кстати, это поясняет, почему методы можно определять сразу внутри класса. Ранее было упомянуто, что **класс в середине его определения всё ещё считается incomplete type**. Кажется, это должно запретить нам использовать класс в определении его методов. Но методы разбираются после класса, потому нет.

А вообще обычно пишут объявления функций в *class.h*, а определения в *class.cpp*. Если определение функции сделано внутри класса, то она неявно помечается как `inline`, но мы не всегда хотим этого (дольше время компиляции из-за зависимостей).

```c++
// struct.h:
struct complex {
	void conjugate();
private:
	double re;
	double im;
};
```
```c++
// struct.cpp:
void complex::conjugate() {
	im = -im;
}
```

## Права доступа.
А в чём глобально разница между внешней функцией и методом? Ну, во внешнюю функцию можно передать `nullptr`, но это легко исправляется ссылками (см. дальше), они тоже не бывают `nullptr`. А вот что действительно важно — права доступа. Как и в Java, вы можете показывать и скрывать поля и методы класса ключевыми словами `public` и `private` соответственно: к `public` полям и методам могут обращаться все вообще, а к `private` — только методы того же класса. Ещё есть модификатор `protected`, он тоже как в Java, и о нём [попозже](./07_inheritance.md#protected). Итого такой код:
```c++
struct complex {
private:
	double re;
	double im;

public:
	void conjugate() {
		im = -im;
	}
};
```
Скомпилируется, а если сделать `conjugate` внешней функцией — то нет.

### Что должно быть `private`, а что `public`? Инварианты класса.
На кой нам вообще `private`? А вот есть у вас двоичное дерево:
```c++
struct node {
	node* left;
	node* right;
	node* parent;
	int value;
};
```
Но двоичное дерево — это же не только вот это, а ещё и набор условий (из серии `this->right->parent == this` или что все значения слева меньше текущего). Если любой лох может изменить поля любым образом, то ситуация будет вырисовываться довольно грустной. Поэтому у нас и есть `private`, который не позволяет всяким лохам это делать. Но на самом деле не только для этого, об этом чуть позже. А пока промежуточный вывод: **то, что может испортить инвариант класса, должно быть `private`.**

Условия, истинность которых считается эквивалентной корректности класса называются *инвариантом класса*. Инварианты класса, кстати, не всегда очевидны. Вот есть у вас дробь:
```c++
struct rational {
	int32_t num;
	int32_t denom;
};
```
Какой у неё инвариант? А вот есть два варианта:
- `denom != 0`.
- `denom > 0 && gcd(num, denom) == 1`.

Самое грустное, что оба варианта верны — в зависимости от выбора изменятся реализации функций. Например, в случае `denom != 0` будет сложение попроще, а сравнение посложнее. И, что грустно, зачастую понять инвариант вы можете только по коду (а код ещё и ошибки может содержать).

Поэтому **полезно бывает в отладочных целях писать метод `void check_invariant() const`, после чего на тестировании вставлять его до и после каждого публичного метода**. Была даже история о том, как чуваки взяли много проектов по красно-чёрным деревьям из GitHub, повставляли в них проверку инварианта и нашли кучу ошибок. А единственные проекты, где не нашли, уже содержали проверку. А дело в том, что красно-чёрное дерево может не сломаться полностью, если допустить в нём ошибку — вы можете ошибиться так, что оно всё ещё будет деревом поиска, всё ещё сможет добавлять и удалять элементы, но оно будет работать медленнее, чем должно теоретически, потому что будет неправильно покрашено. И в этом случае вам проверка инварианта и поможет.

Но есть грустный момент: **иногда инвариант проверить невозможно**. Скажем, мы пишем контейнер, у него должны быть `capacity` выделенных байт, первые `size` из которых проинициализированы. Ну, фиг мы это проверим. Мы можем теоретически написать какой-то умный аллокатор, который имеет проверку первого, а разломав компилятор сможем проверить второе. Но это никто делать не будет, увы:(

Теперь вернёмся к основному вопросу (для чего `private`) и выделим ещё один аргумент. Можем ли мы делать в классе `complex` поля доступными? Мы можем хранить числа в полярной форме, а вещественную и мнимую часть считать, но это кринж. Есть аргумент получше: в компилятор GCC, например, имеет встроенные комплексные числа через ключевое слово `__complex__`. И использует в реализации `std::complex` он именно их, а не два `double`'а. Поэтому тут уже сто́ит сделать их **`private` — мы даём себе простор для модификации**. 

Проверять инварианты можно ассертами (`assert`), про них в отдельном файле. Будет.

<!--
Не стоит злоупотреблять ими, потому что это дорого, лучше делать это при дебаге и тестировании.

Это противоречит тому, что рассказывали нам. Точнее, "Не стоит злоупотреблять ими". Нам наоборот было сказано, если ваш код не замедляется от assert'ов вдвое, в нём мало assert'ов.
-->

## Конструкторы.

Все его публичные методы класса предполагают, что инвариант выполнен до вызова. Поэтому когда мы создаём новый объект, хотелось бы, чтобы он выполнялся с самого начала. Для этого есть специальные методы — *конструкторы*:
<!--
```c++
struct complex {
	complex(re, im) {
		this.re = re;
		this.im = im;
	}
	complex() {		// конструктор по умолчанию
		this.re = 0;
		this.im = 0;
	}
	void conjugate() {	
		im = -im;
	}
private:
	double re;
	double im;
};

void f(complex);
int main() {
	complex c; // вызывается конструктор по умолчанию
	complex d(1., 2.);
	f(complex(1., 2.));
}
```

Это не очень показательный пример (в смысле, как первый пример) из-за того, что никаких инвариантов тут нет. Поэтому я заменю его на класс с инвариантами.
-->

```c++
struct string {
private:
	char* data;
	size_t size;
	size_t capacity;
	/* Инвариант:
	 * 1) `capacity` байт выделено в `data`.
	 * 2) `size <= capacity` из них заполнено.
	 * 3) `data` — корректный указатель либо `nullptr` (если `capacity == 0`).
	 */

public:
	// Вот это конструктор.
	// Он обеспечивает исполнение инварианта в начале.
	string() {
		data = nullptr;
		size = capacity = 0;
	}
};
```

Конструктор без аргументов называется *конструктором по умолчанию*. **Конструкторы вызываются компилятором (и только им) всегда, когда вы создаёте объект.** Вы можете явно указать, какой конструктор вызвать вот так:
```c++
struct complex {
private:
	double re;
	double im;

public:
	complex() {
		re = im = 0;
	}
	complex(double re, double im) {
		// Так называть параметры можно. Можно и ещё круче, но в разделе про списки инициализации.
		this->re = re;
		this->im = im;
	}
};
void main() {
	complex a1;
	complex a2();
	complex a3 = complex();
	complex a4{};

	complex b1(1, 2);
	complex b2 = complex(1, 2);
	complex b3{1, 2};
}
```
Первые 4 определения равносильны. Как и последние 3. Кстати, выражение вида `complex(1, 2)` может как оно есть в функцию передаваться — тогда создастся временный объект и передастся. Этот временный объект, кстати, является rvalue.

Теперь давайте внимательнее посмотрим на `a3` и `b2`. Там мы вроде как сначала создаём объект, а потом присваиваем его куда-то. Так вот да, но нет. У компилятора есть такое понятие как избегание копирования: если правый аргумент — rvalue, то он ничего не копирует, а просто вызывает конструктор на `a3`/`b2`.

### Неявные конструкторы.
```c++
struct complex
{
private:
	double re;
	double im;

public:
	complex() { /*...*/ }
	complex(double re, double im) { /*...*/ }
	complex(double re) {
		this->re = re;
		this->im = 0;
	}
};

void foo(complex) { /*...*/ }

void main() {
	complex a = 42;
	foo(42.0);
}
```
Такой код неявно преобразует `42.0` в `complex` и вызовет от него функцию. В случае с `complex` это оправдано, но если у вас контейнер инициализируется количеством элементов, то так неявно делать странно. Поэтому если вы такого не хотите, напишите перед конструктором слово `explicit`: тогда вы запретите ещё `complex a = 42`, можно будет только `complex a(42)`.

## Деструкторы.
Помните `struct string`? Там же нам надо освободить данные при удалении объекта. А вот для конструкторов есть парные функции — деструкторы, которые автоматически вызываются, когда объект уничтожается:
```c++
struct string {
private:
	char* data;
	size_t size;
	size_t capacity;

public:
	// ...
	~string() {
		free(data);
	}
}

void foo() {
	string s;
} // Вызовется деструктор `s` при выходе из функции.
```
Когда происходит уничтожение?
- Обычные переменные умирают когда наступает фигурная скобка блока, где вы объявили переменную. Совершенно не важно, каким образом вы покидаете блок, `return` у вас, `break`, `throw` или даже `goto`. Только **если `longjmp` вы используете, тогда вы не знаете, вызовутся деструкторы или нет**. Мораль — **не используйте `longjmp`**, потому что он всё равно корректно работает только вверх по стеку, а вверх по стеку можно заменить на `throw`-`catch`.
- Временные объекты умирают по концу выражения, где они созданы.
- Для глобальных переменных конструктор вызывается до main, а деструктор — после него.
- Для полей класса конструктор вызывается до конструктора этого класса, а деструктор — после его деструктора.

При этом **деструкторы объектов одного блока вызываются в порядке, обратном порядку конструкторов**.

<!--
## Немного про const и указатели

Вот это всё не тут, а там же, где макросы (в компиляции).
Что тут, что там это выглядит не очень, но тут как-то более не к месту.
Но вообще в обсуждении `this` выше хорошо бы уже знать про const указатели, то есть раньше, чем тут.
-->

## Операторы.
Для класса `complex` очень хочется иметь арифметические операции, и не хочется как в какой-то помойной Java называть их `add` или `mul`. Чтобы так можно было, в C++ есть ключевое слово `operator`. Они пишутся как обычные функции, только называются как `operator+`, `operator-` и тому подобное.

Также как и обычные функции операторы могут быть внешними или внутренними (правда, не все):
```c++
complex operator+(complex a, complex b) {
	return complex(a.real() + b.real(), a.imag() + b.imag());
}
```
Или
```c++
class complex {
	// ...
	complex operator+(complex other) const {
		return complex(re + other.re, im + other.im);
	}
};
```
Работают они как совершенно обычные функции, поэтому сказать непосредственно про них можно немногое.

Если вы **пишете оператор, то хотя бы один из его элементов должен быть пользовательским типом** (нельзя переопределить оператор для `int, int`, но можно, например, для `std::vector<int>` и `int`).

Если есть желание почитать поподробнее, то почитайте <!-- К чему тут cppreference вообще? К дополнительному чтению? -->[cppreference](https://en.cppreference.com/w/cpp/language/operators).

### Оператор `->`.
Особо нужно посмотреть на `->`. Его обычно перегружают, когда пишут какие-то свои указатели. И выглядят это вот так:
```c++
struct my_ptr {
	// Что-то.
	complex* operator->() {
		return /* Что-то */;
	}
};
```
Можно было бы подумать, что `->` — это бинарный оператор (у него есть то, у чего мы берём поле/метод и имя этого самого поля/метода). Но правая штука — это не выражение. В C++ нет рефлексии. Поэтому **`->` — это унарный оператор**. Если вы возьмёте `my_ptr x` и вызовете `x->im`, то это преобразуется в `(x.operator->())->im`. Поскольку `operator->` возвращает `complex*`, к нему нормально можно применить `->`. А ещё можно из оператора `->` вернуть что-то другое, к чему применим оператор `->`. И тогда они будут вызываться по цепочке, пока не дойдём до обычного указателя.

## Ссылки.
Операторы подводят нас к вопросу, что делать с `+=` и подобными? Точнее, как перегрузить его вовне класса? Совершенно точно не так:
```c++
void operator+=(complex a, complex b) {
	a.set_real(a.real() + b.real());
	a.set_imag(a.imag() + b.imag());
}
```
Есть идеологически неверное, но всё же рабочее решение:
```c++
void operator+=(complex* a, complex b) {
	a->set_real(a->real() + b.real());
	a->set_imag(a->imag() + b.imag());
}
int main() {
	complex x, y;
	&x += y;
}
```

Сделать такое можно, используя ссылки. О них можно думать, как о константных указателях со специальным синтаксисом:
```c++
T a;                         T a;

T* const ptr = &a;           T& ref = a;
foo(*ptr);                   foo(ref);
ptr->bar;                    ref.bar;
```
Ещё **ссылки нельзя перенаправлять на другие объекты** (раз уж это неизменяемый указатель), и **ссылка не бывает `nullptr`**, (ну, правда, зачем вам константный указатель на `nullptr`).

И теперь мы можем увидеть, как делать нужно:
```c++
complex& operator+=(complex &a, complex b) {
	return a = a + b;
}
```
**Во всех присваиваниях (хоть `=`, хоть `@=`) возвращаемое значение — это левый аргумент**, чтобы было консистентно со строенными типами.

А ещё если мы принимаем экземпляр класса и нам не нужно его менять, можно передавать его по константной ссылке, тогда мы избегаем лишних копирований:
```c++
//       Вот сюда смотреть:     vvvvv        v
complex& operator+=(complex &a, const complex& b) {
// Или так, смысл такой же:     complex const& b
	return a = a + b;
}
```

Если функция принимает `const&`, то в неё можно передавать временный объект (rvalue). Если же она принимает обычную ссылку, то только lvalue. \
Также возвращая по ссылке, возвращаем lvalue, а по значению — rvalue.

## Немного best practices (until C++23).
Сто́ит посмотреть, что делать, если вы реализуете свою строку. Вам хочется оператор `[]`. По-хорошему он выглядит так:
```c++
struct string {
private:
	char* data;
	size_t size;
	size_t capacity;
public:
	// ...
	char& operator[](size_t index) {
		return data[index];
	}
};
```
Но на самом деле вы хотите вызывать этот оператор на неизменяемой строке тоже, а от неё указанный оператор не вызывается (нельзя кастовать `const string* const this` в просто `string* const this`). Поэтому вам придётся написать ещё один вариант этого же оператора:
```c++
	const char& operator[](size_t index) const {
		return data[index];
	}
```
Почему `const char&`, а не `char`? Чтобы от константности строки не зависело, lvalue у вас или rvalue. А **`const char&` — это lvalue, у него есть адрес**.

## Продолжение темы операторов.
### Перегрузка операторов внутри класса и снаружи.

Операторы можно перегружать как функции (снаружи класса) и как методы (внутри класса). Соответственно, у операторов, перегруженных как методы, первым аргументом будет неявный `this`. 

У операторов может срабатывать неявное приведение типов (если есть не `explicit` конструктор). При этом если оператор перегружен как функция, то приводится любой из аргументов, а если как метод, то все кроме первого:

```c++
struct complex {
	// ...
	/* implicit */ complex(double re) { /*...*/ }

	complex operator-(const complex& other) const {
	return complex(re - other.re, im - other.im);
}
};
complex operator+(const complex& a, const complex& b) {
	return complex(a.real() + b.real(), a.imag() + b.imag());
}
void main() {
	complex a;
	a + 42; // ok, operator+(a, 42).
	a - 42; // ok, a.operator+(42).
	42 + a; // ok, operator+(42, a).
	42 - a; // Error, `42.operator-(a)` у int нет методов.
}
```
Мораль: **перегружайте операторы внешне**. Если первый аргумент — это ваш тип, то проблема уже описана выше, а если не ваi, то внутренне его вообще не перегрузить.
```c++
Vector operator*(double d, Vector const& v);
```
Некоторые операторы необходимо перегружать только внутри класса: `(type)`, `[]`, `()`, `->`, `->*`, `=`. С ними не возникает проблем с конверсией первого аргумента (если вы хотите неявно преобразовывать аргумент этих операторов куда-то, что-то вы делаете не так). Но с присваиванием есть еще одна проблема.

<!--
Вот так перегружается оператор `()`, заметьте, что можно сделать это для разного количества аргументов:

```c++
bool operator()(double d) const;
void operator()(double a, double b);
```

Какая-то не самая полезная информация, как мне кажется. РАз уж тут есть ссылка на cppreference.
-->

```c++
struct complex {
	// ...
	complex& operator=(const complex& other) { /*...*/ }
};

complex foo() { /*...*/ }

int main() {
	foo() = complex(1, 2);
}
```
Бредятина какая-то. А дело в том, что никто не знает, что `operator=` должен принимать не временный объект первым аргументом. Если бы можно было перегрузить его внешне, то такой проблемы бы не было (там было бы явно указано `complex& left`). И эта проблема в C++11 правится так:
```c++
struct complex {
	// ...
	complex& operator=(const complex& other) & { /*...*/ }
	//          Вот сюда смотреть:           ^
};
```

### Increment и decrement.
Кстати, надо сразу рассказать, как перегружать `++` и `--`, ведь у вас два таких. Тут синтаксический костыль — **постфиксные операторы принимают второй аргумент `int`, который не используется**. Неиспользуемый — *dummy*. Когда вызывается постфиксный оператор, всегда передаётся аргумент, хотя можно и вручную вызвать оператор как функцию и передать любое значение: `a.operator++(2)`.

```c++
struct big_integer {
     big_integer& operator++() { // prefix
        // ...
        return *this;
     }
     big_integer operator++(int) { // postfix
        big_integer tmp(*this);
        ++(*this);
        return tmp;
     }
};
```

### C-style cast.
```c++
struct string {
	// ...

	// Приведение к `bool`.
	operator bool() const {
		return size_ != 0;
	}

	// Приведение к `char const*`.
	operator char const*() const {
  		if (*this) { // Тут используется приведение к `bool`.
    		return data_;
    	} else {
    		return "";
   	 	}
	} 
}
```
<!--
Аналогично можно перегружать и касты в стиле C++ (static_cast и др.).

Простите, что? Как?
-->

У операторов приведения, как и у конструкторов, можно указывать модификатор `explicit` и запрещать неявное приведение.

### Некоторые ограничения:
<!--
- Оператор `->` должен возвращать указатель или объект класса, для которого он переопределён (по ссылке или по значению).

Это в отдельном разделе.
-->
- Операторы `&&`, `||` при перегрузке теряют своё [специальное поведение](https://en.cppreference.com/w/cpp/language/eval_order) и ведут себя как обычные функции.
- Операторы `+=` и подобные лучше перегружать внутри класса, а `+` снаружи через `+=`. Тогда для `+` будет работать приведение типов. При этом любое присваивание выглядит так: `T& T::operator@=(const T& other) &`;
- Операторы сравнений стоит определять одновременно и согласованно: если определили какой-то один из них, принято определить и все остальные так, чтобы они не противоречили друг другу. При этом принято, чтобы сингнатуры у них были одинаковые. Так, либо ни одна, либо все должны быть `noexcept`.
- Хорошим тоном считается соблюдать стандартный смысл операторов: не перегружать оператор `+` как умножение.
- Приоритет операторов остаётся стандартным.
- Перегружать операторов, которых нет изначально, нельзя.
- Перегружать `::`, `?:`, `.` и `.*` нельзя.
<!--
## Про указатели и массивы

Этот тут какого чёрта вообще делало?
-->

## Special function member функции.
### Копирование и присваивание.
Давайте посмотрим на такой код:
```c++
int main() {
    string s = "Hello";
    string t = s;
}
```
Wait a second, мы же не написали как строки копировать. Почему компилятор их копирует? И самое главное, как? А вот покомпонентно. Если покомпонентно нас не утраивает (а в данном случае не устраивает, потому что мы два раза освободим скопированный указатель), надо написать свой конструктор копирования. И в пару ему оператор присваивания, который легко пишется из предыдущего:
```c++
string& operator=(string other) & {
//Внимание!       ^^^ копия ^^^
    std::swap(other, *this); // Как работает std::swap, оставим за кадром. Пока что.
    return *this;
}
```
Это называется *swap-trick*, и его мы ещё [обсудим в контексте исключений](./08_exceptions.md#swap-trick). Кстати, оператор присваивания также генерируется компилятором. Также покомпонентный. То, что компилятор генерирует сам, называется *специальными функциями-членами класса*. Это:

|         Метод          |        Когда генерируется       |             Что делает по умолчанию             |
|:----------------------:|:-------------------------------:|:-----------------------------------------------:|
|Конструктор по умолчанию|  Не написан ни один конструктор |        Конструирует по умолчанию все поля       |
|       Деструктор       |      Не написан деструктор      |                 Удаляет все поля                |
| Копирующий конструктор |Не написан копирующий конструктор|     Инициализирует объект копиями всех полей    |
|  Оператор присваивания | Не написан оператор присваивания|Присваивает всем полям поля того, что присваиваем|


### `= default;` и `= delete;`.
Есть штуки, которые либо не скопировать. Сетевое соединение, например. Тогда, чтобы явно пометить, что копировать и присваивать класс нельзя, есть вот такой синтаксис:
```c++
my_string& operator=(my_string const&) = delete;
my_string(my_string const&) = delete;
```
Так можно запретить любую специальную функцию-член класса. И можно явно указать, что сгенерированный по-умолчанию вам подходит. Например, можно явно создать конструктор по умолчанию, если есть уже какой-то другой и из-за него по умолчанию не сгенерируется.
```c++
my_string() = default;
```
Так ещё может быть полезно писать, чтобы явно документировать, что функция-член класса вам подходит.

Отличается ли чем-то пустой конструктор от дефолтного? А вот да: **пустой конструктор — это *user-defined*** конструктор. Класс с ***default* конструктором — это  *trivially constructible***. Для них, например, при создании массива не будут вызываться конструкторы.

Если `= default` писать в определении в какой-нибудь из единиц трансляции, то другие единицы трансляции во время компиляции не будут знать, что класс *trivially constructible*.

## Списки инициализации.
Посмотрим вот на что:
```c++
struct string {
    // ...
    string() {
        data = strdup("");
        size = capacity = 0;
    }
    string(const char* str) { /*...*/ }
    string(const string& other) { /*...*/ }
    string& operator=(const string& other) & { /*...*/ }
    ~string() {
        free(data);
    }
};
struct person {
    string name;
    string surname;
    person() {
        name = "Eric";
        surname = "Adams";
    }
};
int main() {
    person p;
}
```
Сколько тут будет аллокаций и деаллокаций памяти? А вот 6. Почему? Давайте аккуратно считать.

0\. Мы вызываем конструктор класса person.\
2. У двух `string`'ов вызывается конструктор по умолчанию, каждый из которых выделает память (`strdup`).\
4. Неявно вызываются конструкторы `string("Eric")` и `string("Adams")`, которые тоже выделяют память.\
6. Два раза выполняется присваивание, которые выделяют и освобождают память.

Можно ли это оптимизировать? Компилятор теоретически может, конечно, но срабатывают эти оптимизации настолько редко и ненадёжно, что надеяться на них нельзя. Как это оптимизировать программисту? При помощи списков инициализации — хочется указать, что мы сразу вызываем конструктор `name` и `surname` от строки, а не по-умолчанию. Это делается при помощи такого синтаксиса:
```c++
struct person {
    string name;
    string surname;
    person() : name("Eric"), surname("Adams")
    {}
};
int main() {
    person p;
}
```
И тут аллокаций будет 2, как и ожидается. Кстати, если есть объект, который не имеет конструктора по-умолчанию, то без списков инициализации просто невозможно.

Что сто́ит сказать про это? А то что списки инициализации — это не только (и не столько) оптимизация, сколько по смыслу не то же самое, что мы написали сначала. Только для встроенных типов разницы особой нет, но вообще списки инициализации обычно и для них используются, потому что это гораздо более идиоматический код. А ещё сто́ит сказать, что **если в списке инициализации не написано ничего для некоторого поля, то для него используется конструктор по-умолчанию**.

Поскольку деструктор у нас один, разрушаются поля в определённом порядке. А в конструкторе вы, вроде как, можете в разном написать. Так вот нет, потому что **список инициализации всегда вызывает конструкторы в порядке расположения полей структуры**. И лучше бы вам писать список инициализации в этом же порядке, чтобы не путать людей. И, кстати, в инициализации некоторого поля можно использовать то, что было создано ранее. Аналогично с использованием `this` в списке инициализации — можно, но осторожно, надо понимать, что конструктор `this` ещё недовыполнился. Аналогично осторожным быть надо в деструкторе по той же причине. Подробнее про порядок инициализации [здесь](https://en.cppreference.com/w/cpp/language/constructor).

Кроме того, конструкторы можно делегировать:

```c++
person() : person("Ivan", "Sorokin") {}
```
Сначала вызовется конструктор от двух `char const*`, а затем будет выполнять тело текущего конструктора.

## Ещё немного best practices.
Давайте вот ещё на что посмотрим. Наш класс `string` вызывает в конструкторе аллокацию памяти. Так вот, вообще считается, что это кринж, так делать не надо.

А ещё при присваивании C-строки в нашу строку мы сначала конструируем, а потом присваиваем, давайте вместо этого явно пропишем `operator=(const char*)`, и будет также меньше аллокаций.

## `new` и `delete`.
Итак, мы обсудили, когда вызываются конструкторы и деструкторы. А если вы хотите вызывать их в произвольный момент, конструируя объект тогда, когда захочет пользователь? Тогда есть `new` и `delete`. Вы пишете `Type* p = new Type(...);` — это конструирует вам объект в куче, а `delete p;` разрушает вам этот объект.

Как это работает? В C есть `malloc` и `free`, они выделяют память на стеке в тот момент, когда вы её попросите (опять же, в рандомный момент). Выделение на куче, кстати, немного дороже, чем на стеке.
```c++
    void* p = malloc(42); // Выделяет 42 байта.
    free(p); // Чистит память, выделенную ранее.
    free(p); // Освобождать то, что уже освободил, нельзя.

    free(nullptr); // Так можно, ничего не произойдет.
```
Двойной `free` вызывает UB или даже [уязвимость](https://awakened1712.github.io/hacking/hacking-whatsapp-gif-rce/).

Разница между ними и `new`—`delete` (упрощённо) в том, что второй вариант гарантирует вызов конструкторов и деструкторов. И в большинстве `new` просто является комбинацией из `malloc`'а и [вызова конструктора на выделенной памяти](https://en.cppreference.com/w/cpp/language/new#Placement_new). Аналогично `delete` вызывает деструктор и `free`.

Ещё есть `new[]` и `delete[]` — менеджмент массива. На всём массиве вызываются конструкторы по умолчанию:
```c++
person* p = new person("Ivan", "Sorokin");
delete p;

person* p = new person[10];
delete[] p;
```

При этом ни в коем случае **нельзя освобождать объект другим способом, нежели он был выделен**. Это в любом случае undefined behaviour, даже если, вроде как, не должно:
```c++
person* p = new person("Ivan", "Sorokin");

p->~person();
free(p);
```