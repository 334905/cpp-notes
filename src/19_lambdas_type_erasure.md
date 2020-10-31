#  Анонимные функции, type erasure, std::function
- [Статья про девиртуализацию](http://hubicka.blogspot.com/2014/01/devirtualization-in-c-part-1.html)
- [Запись лекции №1](https://www.youtube.com/watch?v=ItNnt_7D5w0)
- [Запись лекции №2](https://www.youtube.com/watch?v=bTVLDJjEMlU)
---
## Мотивирующий пример и оптимизации

Какой вариант быстрее?

```c++
bool int_less(int a, int b) {
    return a < b;
}

template <typename T>
struct less {
    bool operator()(T const& a, T const& b) const {
        return a < b;
    }
};

int foo (vector& v) {
    std::sort(v.begin(), v.end(), less<int>());
    std::sort(v.begin(), v.end(), &int_less);
}
```

Оказывается, что первый вариант работает процентов на 30 быстрее на случайных массивах и в 3 раза на отсортированных.

Это происходит за счёт того, что когда в `sort` передаётся объект, на этапе компиляции известно тело функции, которая будет вызываться, и его можно заинлайнить. Когда передаётся указатель на функцию, компилятору не известно, куда он ссылается, поэтому соптимизировать это нельзя.

Если скомпилировать с `-flto`, компилятор может соптимизировать, но только в том случае, если в `sort` передаётся указатель только на одну функцию. При этом с `-O3` может произойти так, что даже если передавать указатели на разные функции, время работы будет одинаковым.

`-flto` уже упоминалось в лекции про компиляцию программ, это ключик, который включает *link time optimization*, смысл которой понятен из названия.

`-O3` добавляет некоторые оптимизации, например, `-fipa-cp-clone`, которая создаёт копию функции и оптимизирует её, если в неё передаётся константа. Почему эти оптимизации не включены по умолчанию? Из-за них может ухудшаться производительность. Например, эта оптимизация может создать много копий функции, что заметно увеличит размер бинарного файла. Почему это плохо? Помимо кэша для данных, у процессора есть кэш инструкций, в таком случае он будет забиваться копиями функции и ухудшаться общую производительность. Это можно не заметить на микробенчмарках, но при сборке всего проекта будет заметно равномерное ухудшение, причина которого неочевидна.

Оптимизация, которая заменяет вызов функции по указателю (или виртуальной функции) на вызов самой функции, называется [девиртуализацией](http://hubicka.blogspot.com/2014/01/devirtualization-in-c-part-1.html).

Помимо этого, девиртуализация может быть "спекулятивной" (в GCC она называется `-fdevirtualize-speculatively`), эта оптимизация генерирует примерно такой код:

```c++
bool int_less(int a, int b) {
    return a < b;
}

void sort(int* a, bool(*comp)(int, int)) {
    // ...
    if (comp == &int_less) {
        int_less(a[4], a[5]);
    } else {
        comp(a[4], a[5]);
    }
}
```

Такая оптимизация может быть выгодной, если `if` находится в цикле и это инвариант, поэтому бранч будет отрабатывать только один раз, а не на каждой итерации.

Такая оптимизация даёт выигрыш, если использовать её с *profile-guided optimization* - техникой оптимизации, которая заключается в запуске программы на каких-то репрезентативных данных и последующей её оптимизации, в зависимости от профайлинга. Например, если чаще всего компаратор это `int_less`, то применится предыдущая оптимизация.

## Анонимные функции

До C++11 при необходимости отсортировать контейнер по кастомному компаратору, нужно было писать структуру или функцию, указатель на которую передаётся. Начиная с C++11, это можно делать следующим образом:

```c++
int main() {
    std::vector<int> v;
    std::sort(v.begin(), v.end(), [](int a, int b) -> bool {
        return a % 10 < b % 10;
    });
}
```

Такая конструкция называется лямбдой. На самом деле, это генерит почти такой же код, как определение структуры и передача её объекта. Компилятор генерит структуру с `operator()`, тип его возвращаемого значения выводится из `return`'ов (как `auto`), но его можно указать и явно (как в примере выше).

Синтаксис у лямбд следующий: в круглых скобках - аргументы, которые принимает функция, в фигурных - тело функции, а квадратные скобки нужны для захвата переменных из контекста (такой объект называется *closure*).

```c++
struct mod_comparer {
    mod_comparer(int mod) : mod(mod) {}
    
    bool operator()(int a, int b) const {
        return a % mod < b % mod;
    }
  private:
    int mod;
}

void foo(int mod) {
    std::vector<int> v;
    std::sort(v.begin(), v.end(), mod_comparer(mod));
    std::sort(v.begin(), v.end(), [mod](int a, int b) -> {
        return a % mod < b % mod;
    });
}
```

Захват в примере выше происходит по значению, захватывать можно и по ссылке. Кроме того, можно захватить весь контекст по значению, а некоторые по ссылке

```c++
int main() {
    int x, y, z;
    [x, &y, z](){}; // y - по ссылке, остальные по значению
    [=](){}; // всё по значению
    [=, &y](){}; // всё по значению, y - по ссылке
    [&](){}; // всё по ссылке
    [&, x](){}; // всё по ссылке, x - по значению
}
```

При захвате всего контекста, `this` захватывается тоже. При этом при захвате всего контекста, захватывается только то, к чему есть обращение в теле лямбды.

Если в теле лямбды, которая объявлена внутри класса, используется `this`, то это `this`, который соответствует структуре, а не внутреннему представлению лямбды. Это происходит из-за того, что все имена резолвятся в коде ещё до создания структуры. 

Смотреть на то, во что разворачиваются лямбды, можно на [C++ insights](https://cppinsights.io/).