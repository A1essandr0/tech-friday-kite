**UDS vs TCP sockets**

Сокет - абстракция на уровне ОС для обмена байтиками между процессами ОС. Если процессы на одной машине, то можно использовать UDS - Unix domain sockets, которые выглядят для нас как файлы. Если процессы на разных машинах, то приходится использовать TCP сокеты, которые выглядят как сетевые порты. На практике мы часто используем TCP сокеты для обмена данными между процессами и на одной машине - через loopback-интерфейс, т.е. локалхост.

Типичный кейс - сервисы, работающие за Nginx. Он работает как реверс-прокси, т.е. принимает запрос извне и перенаправляет его сервису. Перенаправляет через сокет.

```Nginx
upstream fastapiserver_tcp {
	server 127.0.0.1:8020;
}

upstream fastapiserver_uds {
	server unix:/tmp/fastapiserver.sock;
}

server {
	server_name server.name;
	listen 8080;

	location /fastapi_tcp/ {
		include proxy_params;
		proxy_pass http://fastapiserver_tcp/;
	}

	location /fastapi_uds/ {
		include proxy_params;
		proxy_pass http://fastapiserver_uds/;
	}
}
```

У файловых сокетов есть несколько преимуществ перед сетевыми:
1. Процессы, использующие файловые сокеты знают, что они на одной машине, и не делают различных проверок, св. с сетевыми эффектами (в первую очередь кодирование/декодирование tcp пакетов). Поэтому работают немного эффективнее.
2. Процессы знают pid-ы друг друга, кто их запустил и т.д.
3. Можно управлять доступом к сокету через пермиссии файла.
4. С файл-сокетами нет опасности наткнуться на занятый порт.
5. Нет возможности по недосмотру оставить порт открытым наружу,  увеличить поверхность для атаки.

Вроде бы этого достаточно для того, чтобы не использовать TCP сокеты для общения процессов на одной машине. Но не все так просто.


**FastAPI**
Для запуска питоновского приложения на FastAPI мы используем Nginx -> uvicorn -> FastAPI. Собственно сервером тут выступает Uvicorn, он же создает несколько воркеров, каждый из которых обслуживает инстанс приложения FastAPI. Именно между Nginx и Uvicorn идет общение через сокет.

```Python
import uvicorn # 0.31.1
from fastapi import FastAPI # 0.115.2
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/")
async def hey():
	return JSONResponse({"msg": "hey yourself", "from": "fastapi"})

if __name__ == '__main__':
	uvicorn.run("fastapi_uvicorn:app", uds='/tmp/fastapiserver.sock', workers=4)
```

```Bash
#!/bin/bash
uvicorn fastapi_uvicorn:app --uds /tmp/fastapiserver.sock --workers 4
```

```Bash
#!/bin/bash
curl --unix-socket  /tmp/fastapiserver.sock http://127.0.0.1
```
В Uvicorn изначально была поддержка сокетов. Но до какого-то времени их нельзя было использовать в случае нескольких воркеров:
https://github.com/encode/uvicorn/issues/428
При этом люди натыкались на эту проблему уже в процессе использования. Это очередное напоминание о том, чтобы поменьше доверять документации, и побольше исходникам. 


**Django**
Для Django цепочка будет Nginx -> gunicorn -> Django. В данном случае сервером / менеджером процессов выступает gunicorn. Этот стек значительно старше, и в нем традиционно использовали именно файловые сокеты.
```Bash
#!/bin/bash
gunicorn --workers=4 --bind unix:/tmp/djangoserver.sock db_app.wsgi:application
```


**aiohttp**
Для приложения, в котором вебсервер является не основной, а вспомогательной задачей, часто удобно использовать легковесный сервер. Например, если приложение активно использует веб-клиент aiohttp, то логично использовать aiohttp и как сервер тоже, не плодя зависимости.
```Python
import asyncio
from aiohttp import web # 3.9.5

async def main():
	app = web.Application()

	async def view_index(request: web.Request):
		return web.json_response({"msg": "hey yourself", "from": "aiohttp"})

	app.add_routes([web.get("/", view_index)])
	
	runner = web.AppRunner(app)
	await runner.setup()
	
	server = web.UnixSite(runner, path='/tmp/aiohttpserver.sock')
	await server.start()

	try:
		while True:
			await asyncio.sleep(5)
	except asyncio.CancelledError:
		await app.cleanup()


if __name__ == '__main__':
	asyncio.run(main())
```

Так как в этом случае нагрузки на сервер не ожидается, то несколько воркеров не нужны, как и менеджер для них.
(При очень большом желании можно запустить несколько процессов вручную и использовать Nginx как балансировщик; в этом случае важно не забыть про пермиссии файлам сокетов):
```Nginx
upstream aiohttpserver_uds {
	server unix:/tmp/aiohttpserver_1.sock;
	server unix:/tmp/aiohttpserver_2.sock;
	server unix:/tmp/aiohttpserver_3.sock;
}

server {
	server_name server.name;
	listen 8080;

	location /aiohttp_uds/ {
		include proxy_params;
		proxy_pass http://aiohttpserver_uds/
	}
}
```


**Docker**
Для того, чтобы использовать файловый сокет, вроде бы можно просто смонтировать volume с соответствующим .sock файлом. Но это противоречит принципам cloud native: контейнер вообще не должен знать, где (на какой машине) и кем (каким оркестратором) он запускается. Все, что у него есть, это переменные окружения. Поэтому контейнерам естественно и привычно использовать именно сетевые интерфейсы для общения между собой.

Кстати, в случае cloud native есть вопрос о воркерах. Если мы запускаем внутри одного контейнера несколько воркеров, например, как мы делали 4 воркера через uvicorn, то каждый из них это отдельный процесс ОС с отдельно выделенной памятью. И если наш ворклоад интенсивно потребляет память, например, использует ML-модель размером в 1гб, то 4 воркера это 4гб и перспектива для контейнера умереть по out-of-memory. Гораздо естественнее тут было бы поднять 4 контейнера и внутри каждого иметь ровно 1 воркер.


**Итог**
Главный вывод получился несколько неожиданным для меня. 
Unix-сокеты - вещь хорошая и вроде бы удобная. Но устаревшая. Пока что не сильно, но уже безнадежно. Знать полезно, использовать вряд ли.
