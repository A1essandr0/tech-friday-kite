# 26.04.2024 - Architectures 2 + пара моментов по инфраструктуре

**Питоновский logging**

Общий setup:

[https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/logger.py?ref_type=heads](https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/logger.py?ref_type=heads)

1. Создавать логгер для каждого файла - плохая идея (логически логгер относится не к файлу, а к некоторому компоненту приложения)

```python
logger = logging.getLogger(__name__)
```

Использовать корневой логгер - плохая идея (в результате все логгеры вообще смешиваются в одну кучу)

```python
logging.info("MSG")
```

Лучше создавать в одном месте все нужные логгеры c их конфигурациями, а затем инжектить (наилучший вариант, хотя у питонистов не принято) или хотя бы импортить:

```python
logging.config.dictConfig(config=logging_config)
root_logger = logging.getLogger()
app_logger = logging.getLogger("app")
kafka_logger = logging.getLogger("kafka") 
test_logger = logging.getLogger("test")
```

```python
from libs.loggers import app_logger as logger, kafka_logger as k_logger
```

Библиотеки часто используют свои логгеры, и их можно/нужно конфигурировать, например:

[https://gitlab.com/kite9/kgrightsmanager/-/blob/dev/app/bot.py?ref_type=heads#L39](https://gitlab.com/kite9/kgrightsmanager/-/blob/dev/app/bot.py?ref_type=heads#L39)

```python
logging.getLogger("httpx").setLevel(logging.WARNING)
```

Проверил производительность создания нескольких логгеров vs использование одного, разница совсем небольшая: [https://gist.github.com/A1essandr0/00d49ac512b0d796db210fadbf65309b](https://gist.github.com/A1essandr0/00d49ac512b0d796db210fadbf65309b)

```python
def check_performance():
    with cProfile.Profile() as pr:
        multiple_loggers(n_loggers=200, n_messages=200)
        single_logger(n_messages=40000)

    pr.print_stats(sort='tottime')

# 1.635 performance.py:30(multiple_loggers)
# 1.601 performance.py:41(single_logger)
```

1. неплохо бы иметь возможность логировать в json

[https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/json_formatter.py?ref_type=heads](https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/json_formatter.py?ref_type=heads#L6)

Подробные объяснения, в т.ч. c подробностями по поводу производительности (синхронность логирования и треды): [https://www.youtube.com/watch?v=9L77QExPmI0&ab_channel=mCoding](https://www.youtube.com/watch?v=9L77QExPmI0&ab_channel=mCoding)

---

**Конфигурация через pydantic_settings** 

У config_loader есть большая проблема - поля конфигурации не типизированные, т.к. это просто словарь.

Кроме того, 

- он не поддерживает дефолтные значения
- не умеет (нормально) работать с json
- не умеет работать с toml (который, между нами говоря, прекрасен),
- у проекта на гитхабе целых 8 звезд https://github.com/adblair/configloader/

Я давно перешел на [pydantic_settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) / https://github.com/pydantic/pydantic-settings. Он уже с того времени перестал быть частью pydantic и приходится устанавливать отдельно.

Чтобы не ломало при переходе:

```python
from libs.settings import Settings as Config
```

Settings/Config это pydantic модель, в которой явно задаются поля, их типы, дефолтные значения

[https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/settings.py?ref_type=heads#L59](https://gitlab.com/kite9/kitetemplates/-/blob/main/fastapi_service/app/libs/settings.py?ref_type=heads#L59)

```python
class Settings(BaseSettings):
    model_config=SettingsConfigDict(
        env_prefix=f"{SERVICE_NAME}_",
        case_sensitive=False,
    )
    
    KAFKA_CLIENT_TYPE: str = "mock"
    KAFKA_BROKER: str = "localhost:9093"
    KAFKA_USERNAME: str = "username"
    KAFKA_PASSWORD: str = "password"
```

```python
broker = config.get("KAFKA_BROKER") # можно ошибиться в имени настройки
broker = config.get(config_loader.KAFKA_BROKER) # нужно заполнять еще и текст
...
broker = config.KAFKA_BROKER # IDE подсказывает все возможные поля
```

Конфигурируем порядок вычитывания конфигурации (сначала дефолтные значения, потом окружение, потом yml)

```python
    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        """
        Define the sources and their order for loading the settings values.

        Args:
            settings_cls: The Settings class.
            init_settings: The `InitSettingsSource` instance.
            env_settings: The `EnvSettingsSource` instance.
            dotenv_settings: The `DotEnvSettingsSource` instance.
            file_secret_settings: The `SecretsSettingsSource` instance.

        Returns:
            A tuple containing the sources and their order for loading the settings values.
        """
        return (
            init_settings, 
            env_settings,
            YamlConfigSettingsSource(settings_cls, yaml_file="./config.yml"), 
        )
```

И наслаждаемся.

Например, на уровне класса, который вычитывает YAML конфиг, я реализую механику, которая позволяет вычитывать из yml списковые значения. Можно также это делать через Field.

Т.е. если в конфиге, который нужен ансиблу есть вот такое:

```python
DIRECTORS_IDS: "4411,4412,4413"
```

То по результату вычитывания оно превратится в:

```python
DIRECTORS_IDS: list[int] = [4411, 4412, 4413]
```

---

**Переиспользование сессий aiohttp**

[https://gist.github.com/A1essandr0/a32bfaa3e5bdb5ea7310cc0664d37176](https://gist.github.com/A1essandr0/a32bfaa3e5bdb5ea7310cc0664d37176)

```python
class HttpClient:
    connector: aiohttp.TCPConnector = None
    session: aiohttp.ClientSession = None

    def __init__(self):
        self.connector = aiohttp.TCPConnector(
            # default parameters
            keepalive_timeout=30,
            limit=100, 
            limit_per_host=0,
        ) 
        self.session = aiohttp.ClientSession(connector=self.connector)

    async def get_with_recycling_session(self, **kwargs):
        return await self.session.get(**kwargs)

    async def get_with_new_session(self, **kwargs):
        async with aiohttp.ClientSession() as session:
            return await session.get(**kwargs)
```

Когда я запускал в первый раз, для [https://gosuslugi.ru](https://www.gosuslugi.ru/) получалась разница во времени 1,5-2 раза на всего 200 реквестах.

Вчера запустил через мобильный интернет:

time for GET variant with new sessions: 32.53415385799963,  okays: 200 of 200
time for GET variant with recycling sessions: 5.379521050000221,  okays: 200 of 200

Т.е. чем хуже соединение, тем драматичнее может быть эффект.