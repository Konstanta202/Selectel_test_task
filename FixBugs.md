 # Отчет по отладке приложения Selectel Vacancies API
 # Скалисусов Константин Геннадьевич

## 1. Ошибка Pydantic в конфигурации при первичном запуске
**Проблема:**
При первом запуске сервера (app-1) возникала ошибка валидации Pydantic.

в файле app/core/config на 14 строке - validation_alias

Код до:
    database_url: str = Field(
        "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
        validation_alias="DATABSE_URL",
    )

**Ошибка:**

app-1  | pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
app-1  | database_url
app-1  |   Extra inputs are not permitted [type=extra_forbidden, input_value='postgresql+asyncpg://pos...stgres@db:5432/postgres', input_type=str]
app-1  |     For further information visit https://errors.pydantic.dev/2.12/v/extra_forbidden

**Исправление:**
Переименование DATABSE_URL на DATABASE_URL


## 2. Ошибка парсинга данных с API

**Проблема:**
При запуске парсера возникала ошибка 
AttributeError: 'NoneType' object has no attribute 'name'

В файле app/services/parser.py на 44 строке - "city_name"

Как было:
    "city_name": item.city.name.strip() 

**Ошибка:**
AttributeError: 'NoneType' object has no attribute 'name'
    File "/app/app/services/parser.py", line 43, in parse_and_store
        "city_name": item.city.name.strip(),

**Причина:**
Когда приходят вакансии с полем city: null, а код пытается обратиться к атрибуту
name у None

**Исправление:**
Добавление дополнительных проверок если поле null
"city_name": item.city.name.strip() if item.city and item.city.name  else None


## 3. Странная зависимость в requirements.txt

**Проблема:**

В файле requirements.txt подозрительная зависимость 

fastapi==999.0.0; python_version < "3.8"

**Причина:**

Не существует такой версии

**Предположение:**

Такая запись нужна для предотвращения установки версии python ниже 3.8


## 4. Несоответсвие работы фонового парсинга вакансий

**Проблема:** Парсер запускается каждые 5 секунд вместо 5 минут

Из логов видно, что интервал во времени 5 секунд:

app-1  | 2026-02-27 02:38:15,980 | INFO | apscheduler.executors.default | Running job "_run_parse_job (trigger: interval[0:00:05], next run at: 2026-02-27 02:38:20 UTC)" (scheduled at 2026-02-27 02:38:15.977148+00:00)
app-1  | 2026-02-27 02:38:15,981 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-02-27 02:38:16,441 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-02-27 02:38:16,463 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-02-27 02:38:16,464 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:00:05], next run at: 2026-02-27 02:38:20 UTC)" executed successfully
app-1  | 2026-02-27 02:38:20,983 | INFO | apscheduler.executors.default | Running job "_run_parse_job (trigger: interval[0:00:05], next run at: 2026-02-27 02:38:25 UTC)" (scheduled at 2026-02-27 02:38:20.977148+00:00)
app-1  | 2026-02-27 02:38:20,983 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-02-27 02:38:21,346 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-02-27 02:38:21,367 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-02-27 02:38:21,368 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:00:05], next run at: 2026-02-27 02:38:25 UTC)" executed successfully


**Причина:**

В функции create_scheduler указан неправильный парамет

Код до:

    def create_scheduler(job: Callable[[], Awaitable[None]]) -> AsyncIOScheduler:
        scheduler = AsyncIOScheduler()
        scheduler.add_job(
            job,
            trigger="interval",
            seconds=settings.parse_schedule_minutes,
            coalesce=True,
            max_instances=1,
        )
        return scheduler

**Исправление :**

Замена seconds на minutes

Логи после исправления

app-1  | 2026-02-27 01:01:22,514 | INFO | apscheduler.executors.default | Running job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:06:22 UTC)" (scheduled at 2026-02-27 01:01:22.505341+00:00)
app-1  | 2026-02-27 01:01:22,516 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-02-27 01:01:22,966 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-02-27 01:01:22,996 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-02-27 01:01:22,996 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:06:22 UTC)" executed successfully
app-1  | 2026-02-27 01:06:22,515 | INFO | apscheduler.executors.default | Running job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:11:22 UTC)" (scheduled at 2026-02-27 01:06:22.505341+00:00)
app-1  | 2026-02-27 01:06:22,518 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-02-27 01:06:22,997 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-02-27 01:06:23,038 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-02-27 01:06:23,040 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:11:22 UTC)" executed successfully
app-1  | 2026-02-27 01:11:22,511 | INFO | apscheduler.executors.default | Running job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:16:22 UTC)" (scheduled at 2026-02-27 01:11:22.505341+00:00)
app-1  | 2026-02-27 01:11:22,514 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-02-27 01:11:23,048 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-02-27 01:11:23,083 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-02-27 01:11:23,084 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:05:00], next run at: 2026-02-27 01:16:22 UTC)" executed successfully


## 5. Неправильный HTTP статус при дубликате

**Проблема:** 
В эндпоинте @router.post, при попытке создать уже существующую запись возвращался статус 200 OK

В файле: app/api/v1/vacancies.py на 51 строке

Код до:

    if existing:
        return JSONResponse(
            status_code=status.HTTP_200_OK,
            content={"detail": "Vacancy with external_id already exists"},
        )

**Исправление:**

Код после:
    if existing:
        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT,
            content={"detail": "Vacancy with external_id already exists"},
        )


## 6. Отсутствие status_code в PUT-методе

**Проблема:** в эндпоинте @router.put,
Не указан явно статус-код

В файле: app/api/v1/vacancies.py на 59 строке

Код до:

    @router.put("/{vacancy_id}", response_model=VacancyRead)

**Исправление:**

Код после:

    @router.put("/{vacancy_id}", response_model=VacancyRead, status_code=status.HTTP_200_OK)


## 7. Опечатка в названии базы данных

**Проблема:** В Settings указано неправильно URL БД по умолчанию

в файле app/core/config.py на 13 строке

Код до:

    "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo"

**Ошибка:**

при запуске без указания БД в .env файле

app-1  | asyncpg.exceptions.InvalidCatalogNameError: database "postgres_typo" does not exist

**Исправление:**

код после:

    "postgresql+asyncpg://postgres:postgres@db:5432/postgres"


## 8. Утечка HTTP-клиента

**Проблема:**

HTTP-клиент не закрывается после использования, что приводит к утечке соединений

в файле app/services/parser.py на 31 строке

код до:
    client = httpx.AsyncClient(timeout=timeout)

**Исправление:**

открытие через async with

код после:

    async with httpx.AsyncClient(timeout=timeout) as client:


## 9. В docker-compose.yml используется запускаются миграции и uvicorn

При использовании command в docker-compose файле для app, при этом команда из 
Dockerfile не запустится, так как в docker-compose идет переопределение 


## 10. 500 ошибка при обновлений данных вакансии

**Проблема:**

При обновлении данных вакакансии с указание поля external_id, которое 
существует у другого пользователя, возникает ошибка 500

**Причина:**

в файле app/api/v1/vacancies.py 
Нету проверки в методе PUT нету проверки на существование такого external_id
в базе данных у другой вакансии 

**Исправление:**

Добавление проверки на присутсвие этого поля и проверки, если это поле не схоже с external_id вакансией, которому мы изменяем данные

    if payload.external_id is not None and vacancy.external_id != payload.external_id:
        existing = await get_vacancy_by_external_id(session, payload.external_id)
        if existing:
            return JSONResponse(
                status_code=status.HTTP_409_CONFLICT,
                content={"detail": "Vacancy with external_id already exists"},
            )


## 11. Потеря данных поля при обновлении данных

**Проблема:**
При обновлении данных, если не указывать поля, которые необходимо обновить
то они станут не теми, которые заданы в схеме модели 

в файле app/crud/vacancy.py на 50 строке 

    for field, value in data.model_dump().items():

**Причина:**

Не указан параметр exclude_unset=True

**Исправление:**

    добавление параметра exclude_unset=True в метод model_dump()



## 12. В файлах используется одинаковая функция для получение сессии

**Исправление:**

1 Создан файл с зависимостями и добавлена функция для получения сессии
2 Добавлен AsyncGenerator для AsyncSession


## 13. Ошибка 500 при создании вакансии с большим external_id

**Проблема:**

При создании вакансии с external_id больше значения int выпадает 500 ошибка

**Варианты решения:**

1 Если значение external_id превышать значение int, то тогда нужно указать в модели, что значение будет BigInteger


2 Если планируемое значение не превышает значение int, то необходимо обработать значение так, чтобы пользователь не мог ввести значение больше int


## 13. Использование global для планировщика

**Проблема:** 
Использует global для хранения ссылки на планировщик
в файле app/main.py

**Потенциальная ошибка:** 
Если где-то случайно изменить _scheduler, то сломается планировщик

**Исправление:** 

Использовать app.state вместо global для:

    app.state.scheduler = create_scheduler(_run_parse_job)