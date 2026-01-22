# ADR 0002: Event Model — In-Process Event Bus

## Статус
Принято

## Контекст
Для модульного монолита требуется способ взаимодействия модулей без прямых зависимостей. В MVP важно:
- Чёткие границы между модулями
- Возможность вынести модуль в микросервис без изменения контрактов
- Простота реализации и отладки

## Решение
Используем **событийную модель** с in-process event bus на старте.

### Основные события системы

1. **experience.created**
   - Публикуется: модулем `experience` при создании raw опыта
   - Подписчики: `nlp_pipeline` (для обработки), `policy` (для проверки приватности)
   - Payload: `user_id`, `experience_id`, `raw_text`, `created_at`

2. **experience.indexed**
   - Публикуется: модулем `nlp_pipeline` после обработки и индексации
   - Подписчики: `retrieval` (для обновления индексов), `answer` (опционально для кэширования)
   - Payload: `experience_id`, `domain`, `attributes`, `embedding_ready`, `indexed_at`

3. **recommendation.generated**
   - Публикуется: модулем `answer` при генерации рекомендации
   - Подписчики: `outcome` (для отслеживания использования), `policy` (для аналитики)
   - Payload: `user_id`, `question`, `recommendation_id`, `sources`, `generated_at`

4. **outcome.submitted**
   - Публикуется: модулем `outcome` при фиксации результата
   - Подписчики: `social_graph` (для обновления весов доверия), `answer` (для улучшения качества)
   - Payload: `user_id`, `recommendation_id`, `outcome_type`, `feedback_text`, `submitted_at`

### Реализация

**In-process Event Bus:**
```python
# app/common/eventbus/bus.py
class EventBus:
    def __init__(self):
        self._subscribers: Dict[Type[Event], List[Callable]] = {}
    
    def subscribe(self, event_type: Type[Event]):
        def decorator(handler: Callable):
            # Регистрация подписчика
            pass
        return decorator
    
    async def publish(self, event: Event):
        # Вызов всех подписчиков синхронно/асинхронно
        pass
```

**События как контракты:**
- Все события определены в `packages/contracts/events.py`
- Используют Pydantic для валидации
- Immutable (dataclass или frozen Pydantic model)

### Пример использования

```python
# modules/experience/service.py
from packages.contracts.events import ExperienceCreated
from app.common.eventbus.bus import event_bus

class ExperienceService:
    async def create_experience(self, user_id: int, text: str):
        # Создание опыта в БД
        experience = await self.repository.create(...)
        
        # Публикация события
        await event_bus.publish(
            ExperienceCreated(
                user_id=user_id,
                experience_id=experience.id,
                raw_text=text,
                created_at=datetime.utcnow()
            )
        )

# modules/nlp_pipeline/service.py
from packages.contracts.events import ExperienceCreated
from app.common.eventbus.bus import event_bus

@event_bus.subscribe(ExperienceCreated)
async def handle_experience_created(event: ExperienceCreated):
    # Обработка без прямого импорта модуля experience
    structured = await process_experience(event.raw_text)
    await index_experience(event.experience_id, structured)
```

## Альтернативы, которые рассматривались

### Прямые вызовы между модулями
**Минусы:**
- Жёсткие зависимости
- Невозможно вынести модуль без рефакторинга
- Циклические зависимости

### Внешний message broker (RabbitMQ/Kafka)
**Плюсы:**
- Готов к масштабированию
- Надёжность доставки

**Минусы:**
- Избыточно для MVP
- Усложняет отладку
- Дополнительная инфраструктура

### Saga Pattern
**Минусы:**
- Слишком сложно для MVP
- Можно добавить позже при необходимости

## Последствия

### Позитивные
- Чёткие границы модулей
- Легко тестировать изоляцию
- Готовность к миграции на внешний брокер
- Простая отладка (все в одном процессе)

### Негативные
- Если один обработчик упадёт — может повлиять на других (нужна обработка ошибок)
- Нет гарантии доставки (для MVP приемлемо)

### Риски и митигация
- **Риск:** Обработчики блокируют друг друга
  - **Митигация:** Все обработчики должны быть async, использовать `asyncio.gather` или очередь

- **Риск:** Потеря событий при падении процесса
  - **Митигация:** В будущем добавить Outbox Pattern (см. `app/common/eventbus/outbox.py`)

### Будущее развитие
При переходе к микросервисам:
1. Event bus становится клиентом RabbitMQ/Kafka
2. События сериализуются в JSON
3. Подписчики становятся отдельными сервисами
4. Контракты событий остаются неизменными

## Связанные решения
- ADR 0001 (Stack) — выбор FastAPI как платформы
- TODO: ADR 0003 — обработка ошибок в event handlers
