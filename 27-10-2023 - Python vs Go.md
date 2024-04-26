# 27.10.2023 - Python vs Go для бэкенда

Почему Python

- Высокая скорость разработки ⇒ удобно делать прототипы, proof-of-concept, MVP
- Требования к разработчику минимальные
- Самое обширное и дружелюбное сообщество
- Экспрессивность языка
- Полная поддержка ООП
- Используется для ML

Почему Go

- Статические типы
- Производительность
    - в 5-50 раз быстрее (в зависимости от юзкейса; для обильного парсинга/сериализации ближе к 50, нежели к 5) [https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/go-python3.html](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/go-python3.html)
    - в ~5-10 раз эффективнее используется память (часто важнее скорости) [https://github.com/A1essandr0/benchmarks/blob/master/docs/memory.md](https://github.com/A1essandr0/benchmarks/blob/master/docs/memory.md)
- Отличная модель конкуррентности
- Удобство сборки
    - отличный стандартизированный toolchain
    - приложение = один статически линкованный бинарник
- Минимализм языка ⇒ предсказуемость и читаемость
- Используется для high-load ⇒ встречаются задачи поинтереснее CRUDов