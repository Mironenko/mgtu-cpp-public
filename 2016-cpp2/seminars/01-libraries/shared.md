# Динамическая библиотека

## Назначение
* Вынесение общей функциональности нескольких приложений в отдельный исполняемый файл.
* Поддержка механизма подключаемых модулей -- плагинов.
* Распространение проприетарного бинарного кода в модулях, не зависящих от используемого компилятора (и, потенциально, от версии рантайма)

---

## Использование

### Декларация использования приложением
* Динамическая библиотека, в отличие от статической, на этапе линковки не объединяется с бинарным кодом программы. Линкер только проверяет, что библиотека содержит символы, требуемые приложению. 

### Загрузка библиотеки
* В случае _динамической компоновки_ библиотека загружается сразу при загрузке приложения.
* В случае _динамической загрузки_ библиотека загружается посредством вызова специализированного API.
* В ходе работы приложения система поддерживает счетчик ссылок на библиотеку из конкретного приложения. При обнулении этого счетчика библиотека выгружается из виртуального адресного пространства приложения.
* При загрузке библиотеки специализированный модуль -- загрузчик -- модифицирует таблицу связывания процедур (PLT) и глобальную таблицу смещений (GOT) -- вносит информацию о внешний функциях и данных.
---

### Динамический загрузчик в linux

* Загрузка библиотеки происходит в момент загрузки приложения. За это ответственен динамический загрузчик.
#### ld-linux.so

Поиск библиотеки:
* LD_PRELOAD -- список объектов, которые будут загружены сразу после загрузки самого приложения
* LD_LIBRARY_PATH -- список путей, которые будут просмотрены в поиске библиотек перед системными
* /etc/ld.so.conf -- конфигурация системных путей поиска библиотек (/usr/lib, /usr/local/lib/)
* Указание нестандартного пути поиска библиотеки при сборке приложения в флаге линкера rpath пути до библиотеки.
```
gcc main.c -lfuncs -Wl,rpath,/opt/my/
```

---

### Динамическая загрузка
Управление загрузкой библиотеки производится через интерфейс функции libdl.so:
```
#include <dlfcn.h>

void *dlopen(const char *filename, int flags);

int dlclose(void *handle);

void *dlsym(void *handle, const char *symbol);

char *dlerror(void);
```

---

#### dlopen flags

* RTLD_LAZY
Perform lazy binding.  Only resolve symbols as the code that references them is executed.  If the symbol is never referenced, then it is never resolved.  (Lazy binding is performed only for function references; references to variables are always immediately bound when the shared object
is loaded.)  Since glibc 2.1.1, this flag is overridden by the effect of the LD_BIND_NOW environment variable.

* RTLD_NOW
If this value is specified, or the environment variable LD_BIND_NOW is set to a nonempty string, all undefined symbols in the shared object are resolved before dlopen() returns.  If this cannot be done, an error is returned.

---

#### dlopen flags

Zero or more of the following values may also be ORed in flags:

* RTLD_GLOBAL
The symbols defined by this shared object will be made available for symbol resolution of subsequently loaded shared objects.

* RTLD_LOCAL
This is the converse of RTLD_GLOBAL, and the default if neither flag is specified.  Symbols defined in this shared object are not made available to resolve references in subsequently loaded shared objects.

* RTLD_NODELETE (since glibc 2.2)
Do not unload the shared object during dlclose().
Consequently, the object's static variables are not reinitialized if the object is reloaded with dlopen() at a later time.

---

#### dlopen flags (continued)

* RTLD_NOLOAD (since glibc 2.2)
Don't load the shared object.  This can be used to test if the object is already resident (dlopen() returns NULL if it is not, or the object's handle if it is resident).  This flag can also be used to promote the flags on a shared object that is already loaded.  For example, a shared object that was previously loaded with RTLD_LOCAL can be reopened with RTLD_NOLOAD | RTLD_GLOBAL.

* RTLD_DEEPBIND (since glibc 2.3.4)
Place the lookup scope of the symbols in this shared object ahead of the global scope.  This means that a self-contained object will use its own symbols in preference to global symbols with the same name contained in objects that have already been loaded.

---

#### Использование dlsym
Объявление указателя на функцию.
```
int f(int a, const char* b, void* c);
typedef int (*f_ptr)(int, const char*, void*);
// ...
f_ptr pf = (f_ptr) dladdr(h_lib, "f");
pf(1, "hello", (void*)0);
```

---

### Обработка загрузки и выгрузки библиотеки

#### C-library

Требуется инициализация глобальных объектов при загрузке библиотеки.
* флаги ld
```
ld ... -init my_init_function ...
gcc -shared ... -Wl,-init,my_init_function ...

ld ... -fini my_finish_function ...
gcc -shared ... -Wl,-init,my_finish_function ...
```

---

* атрибуты функций
```
static void con() __attribute__((constructor));
void con() { printf("I'm called during first dlopen\n"); }

static void dt() __attribute__((destructor));
void dt() { printf("I'm called during last dlclose\n"); }
```

Порядок вызова функций-конструкторов и функций-деструкторов не гарантируется.

---

#### С++ library
При загрузке библиотеки будет осуществлена инициализация глобальных объектов посредством вызова их конструкторов.

---

## Создание динамических библиотек
### Библиотеки функций
#### Объявление экспортируемых функций в C:
```
int my_extern_function()
````
#### Управление атрибутами видимости:

```
__attribute__((__visibility__("default"))) // видимость из всех объектных файлов

__attribute__((__visibility__("hidden"))) // видимость внутри объектного файла
```
Установка значения по умолчанию:
```
gcc -shared ... -fvisibility=hidden ...
```

---

#### Экспортируемые функции в C++:
```cpp
////////my.h/////////////
#ifdef __cplusplus
extern "C" {
#endif

void run();

#ifdef __cplusplus
}
#endif

////////my.cpp/////////////
void run() {
  printString("Hello, world!");
}

```
Причины: несовместимый между компиляторами манглинг имен.

При сборке одним компилятором (одной версией компилятора), можно пренебречь.

---

#### Сборка
```
g++ -shared -fPIC -static-libstdc++ -static-libgcc my.cpp
```

---

### Загрузка библиотек в Windows

#### Порядок поиска библиотеки
1. The directory specified by lpFileName. [IF LOAD_WITH_ALTERED_SEARCH_PATH used]
1. The directory from which the application loaded OR The directory specified by the lpPathName parameter of SetDllDirectory. [IF ANY]
1. The current directory [IF HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\SafeDllSearchMode = false]
1. The system directory. Use the GetSystemDirectory function to get the path of this directory.
1. The 16-bit system directory. There is no function that obtains the path of this directory, but it is searched.
1. The Windows directory. Use the GetWindowsDirectory function to get the path of this directory.
1. The directories that are listed in the PATH environment variable. Note that this does not include the per-application path specified by the App Paths registry key. The App Paths key is not used when computing the DLL search path.

---

##### Изменение путей поиска
```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\SafeDllSearchMode

LoadLibraryEx(..., LOAD_WITH_ALTERED_SEARCH_PATH, ...)

SetDllDirectory()
```

---

### Динамическая компоновка
#### Точка входа в DLL
```cpp
BOOL WINAPI DllMain(
    HINSTANCE hinstDLL,  // handle to DLL module
    DWORD fdwReason,     // reason for calling function
    LPVOID lpReserved )  // reserved
{
    // Perform actions based on the reason for calling.
    switch( fdwReason ) 
    { 
        case DLL_PROCESS_ATTACH:
         // Initialize once for each new process.
         // Return FALSE to fail DLL load.
            break;
        case DLL_THREAD_ATTACH:
         // Do thread-specific initialization.
            break;
        case DLL_THREAD_DETACH:
         // Do thread-specific cleanup.
            break;
        case DLL_PROCESS_DETACH:
         // Perform any necessary cleanup.
            break;
    }
    return TRUE;  // Successful DLL_PROCESS_ATTACH.
}
```

---

##### Ограничения DllMain
_Можно_:
* Использовать функции kernel32.dll (инициализация объектов синхронизации)

_Нельзя_:
* Вызов LoadLibrary из DllMain
* Использование User, Shell, Advapi, COM-функций из DllMain (Может потребовать загрузку библиотеки)

---

###  Динамическая загрузка
```cpp
HMODULE WINAPI LoadLibrary(_In_ LPCTSTR lpFileName);

BOOL WINAPI FreeLibrary(_In_ HMODULE hModule);

FARPROC WINAPI GetProcAddress(_In_ HMODULE hModule, _In_ LPCSTR  lpProcName);
```

---

### Объявление функций
Для экспортируемых из библиотеки функций:
```
__declspec(dllexport) 
```
Для импортируемых в приложение  функций:
```
__declspec(dllimport)
```
Пример:
```
extern "C" __declspec(dllexport) int f(char*);
```

---

### Один заголовочный файл в DLL и приложении
```
#ifdef MYLIBRARY_EXPORTS
#define MYLIBRARY_API __declspec(dllexport) 
#else
#define MYLIBRARY_API __declspec(dllimport) 
#endif

#ifdef __cplusplus
extern "C" {  // only need to export C interface if
              // used by C++ source code
#endif

MYLIBRARY_API int f(char*);

#ifdef __cplusplus
}
#endif
```

---

### Перечисление экспортируемых функций через DEF-файл
При использовании DEF-файла пропадает необходимость указывать __declspec(dllexport)
```
LIBRARY MYLIB
EXPORTS
	function1
	myfunction
	printMe
```
* Project Properties > Linker > Input > Module Definition File
---

### Динамическая компоновка с DLL
* Путь поиска библиотеки при линковке

Project Properties > Linker > General > Additional Library Directories

* Имена линкуемых библиотек (указываются lib-файлы) Project Properties > Input > Additional Dependencies

---

### С++ классы в динамических библиотеках

#### Проблемы
* Отличия в c++ runtime библиотеки и приложения
* Библиотека и приложение используют различные менеджеры памяти
* Манглинг имен может различаться в компиляторе приложения и библиотеки

#### Решения
* Использование динамического runtime (/MD vs /MT, using vcredist)
* Использование одного и того же компилятора при сборке библиотеки и приложения
* Создание c-библиотек и header-only с++ библиотек
* Оперирование объектами библиотеки через интерфейс (абстрактный класс)

---

### С++ классы в динамических библиотеках
```cpp
// public.h
class Polygon {
public:
virtual void draw(Screen* s) = 0;
virtual void move(size_t x, size_t y) = 0;
virtual ~Polygon() = 0;
}

extern "C" Polygon* createPolygon(size_t vertices);
extern "C" void destroyPolygon(Polygon* p);
```

---
### С++ классы в динамических библиотеках
```cpp
// shared.cpp
#include "public.h"
class PolygonImpl : public Polygon {
public:
	void draw(Screen* s) {};
	void move(size_t x, size_t y) {};
	PolygonImpl(size_t vertices) {};
}

Polygon* createPolygon(size_t vertices) {
	return new PolygonImpl(vertices);
}
void destroyPolygon(Polygon* p) { delete p; }
```

---

## Материалы
* http://www.linuxjournal.com/article/3687
* http://www.faqs.org/docs/Linux-mini/C++-dlopen.html
* http://www.codeproject.com/Articles/28969/HowTo-Export-C-classes-from-a-DLL

