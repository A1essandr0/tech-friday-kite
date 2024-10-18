Простое key-value хранилище с интерфейсом словаря. Часть стандартной библиотеки Питона вплоть до 3.12

```Python
import shelve

with shelve.open(filename="storage") as storage:
	storage["key"] = "value"
	another_value = storage["another_key"]

```

- не удаляет значения при перезаписи ключей;
- реализация хранилища зависит от системы, ты не можешь его портировать и даже просто узнать, какую именно реализацию выбирает shelve/dbm - некоторое приключение;
- использует под капотом pickle, который позволяет сериализовать произвольные объекты. Побочное следствие - при десериализации может выполнить произвольный код!

```Python
import os, shelve

class ShellCode:
	def __reduce__(self):
		return (os.system, ('ls -la',))
		# return (os.system,(f"rm '{__file__}'",))

with shelve.open("./ms") as storage:
	storage["code"] = ShellCode()

with shelve.open("./ms") as storage:
	s = storage["code"]
```

Вот так, например, скрипт радостно удалит сам себя.

А можно удалить что-нибудь другое.
А можно порт пробросить.
А можно ключи слить в неизвестном направлении.