# Структура репозитория Trust

## Обзор

Репозиторий организован как **модульный монолит** с чёткими границами между модулями. Каждый модуль готов к выносу в микросервис без изменения контрактов.

## Структура директорий

```
trust/
├── docs/                    # Документация проекта
│   ├── repo-structure.md   # Этот файл
│   └── adr/                # Architecture Decision Records
│
├── apps/
│   └── api/                # Единственное backend-приложение (FastAPI)
│       └── app/
│           ├── main.py     # Точка входа FastAPI
│           ├── settings.py # Конфигурация приложения
│           │
│           ├── common/     # Общая инфраструктура
│           │   ├── db.py          # Подключение к БД
│           │   ├── logging.py     # Настройка логирования
│           │   └── eventbus/      # Event bus реализация
│           │       ├── bus.py     # In-process pub/sub
│           │       └── outbox.py  # Outbox pattern (TODO для будущего)
│           │
│           └── modules/    # Бизнес-модули
│               ├── identity/          # Аутентификация и пользователи
│               ├── social_graph/      # Граф друзей и доверия
│               ├── experience/        # Фиксация опыта (raw + structured)
│               ├── nlp_pipeline/      # Обработка опыта (домен, атрибуты, embeddings)
│               ├── retrieval/         # Поиск по векторной БД
│               ├── answer/            # Генерация ответов
│               ├── outcome/           # Фиксация исходов
│               └── policy/            # Политики приватности и доступа
│
└── packages/
    └── contracts/          # Общие контракты между модулями
        ├── dtos.py        # Data Transfer Objects (Pydantic)
        ├── events.py      # События системы
        └── errors.py      # Общие исключения
```

## Правила зависимостей

### Запрещённые зависимости

1. **Модули НЕ могут импортировать друг друга напрямую**
   - ❌ `from modules.experience import ...` в модуле `answer`
   - ✅ Использовать события через event bus

2. **Модули НЕ могут обращаться к БД других модулей напрямую**
   - Каждый модуль работает только со своими таблицами
   - Межмодульные запросы через события или контракты

### Разрешённые зависимости

1. **Модули → `packages/contracts`**
   - Любой модуль может импортировать DTOs, события, ошибки

2. **Модули → `common/`**
   - Общая инфраструктура (DB, logging, event bus)

3. **`apps/api/app/main.py` → все модули**
   - Только main.py может регистрировать роутеры модулей

## Структура модуля

Каждый модуль должен содержать:

```
modules/{module_name}/
├── api.py         # FastAPI router (endpoints)
├── service.py     # Бизнес-логика
├── models.py      # Pydantic модели для API
└── repository.py  # Доступ к БД (если нужен)
```

### Пример модуля

```python
# modules/identity/api.py
from fastapi import APIRouter
from .service import IdentityService
from .models import UserCreate, UserResponse

router = APIRouter(prefix="/identity", tags=["identity"])

@router.post("/users", response_model=UserResponse)
async def create_user(data: UserCreate):
    # Использует service, не обращается к другим модулям напрямую
    pass
```

## Event bus как граница модулей

Модули взаимодействуют через события:

```python
# Модуль A публикует событие
from packages.contracts.events import ExperienceCreated
from app.common.eventbus.bus import event_bus

await event_bus.publish(ExperienceCreated(user_id=..., experience_id=...))

# Модуль B подписывается на событие
@event_bus.subscribe(ExperienceCreated)
async def handle_experience_created(event: ExperienceCreated):
    # Обработка без прямого импорта модуля experience
    pass
```

## Принципы организации кода

1. **Один файл = одна ответственность**
   - Короткие файлы (< 300 строк)
   - Понятные имена

2. **TODO вместо предположений**
   - Если что-то не описано в architecture.md — оставляем TODO
   - Не придумываем без явного указания

3. **Простота и читаемость**
   - Предпочитаем простые решения
   - Код должен быть самодокументируемым

## Миграция к микросервисам

При необходимости выноса модуля в микросервис:

1. Контракты (`packages/contracts`) остаются общими
2. События становятся сообщениями (RabbitMQ/Kafka)
3. API endpoints становятся HTTP endpoints
4. Бизнес-логика (`service.py`) не меняется

Этот подход обеспечивает плавный переход без рефакторинга ядра.
