
# Паттерны проектирования Business Metrics Analyzer

## 1. Выбор и обоснование паттернов

Для приложения Business Metrics Analyzer выбраны паттерны, обеспечивающие разделение бизнес-логики, доступа к данным и пользовательского интерфейса. Паттерны соответствуют слоистому монолиту с элементами Clean Architecture.

| Паттерн | Где применяется | Обоснование | Альтернативы |
|---|---|---|---|
| Repository | Доступ к данным о продажах и пользователях | Изолирует бизнес-логику от SQL-запросов, упрощает тестирование | Прямые SQL-запросы, непосредственное использование ORM |
| Unit of Work | Импорт данных и операции записи | Обеспечивает атомарность транзакций, фиксацию изменений и откат при ошибках | Ручное управление транзакциями |
| Service Layer | Расчёт KPI, импорт продаж, формирование отчётов | Выделяет бизнес-сценарии в отдельные сервисы, разгружает контроллеры | Размещение логики непосредственно в контроллерах |
| Factory | Создание отчётов Excel и PDF | Позволяет добавлять новые форматы отчётов без изменения вызывающего кода | Условные операторы if/else |
| Strategy | Расчёт выручки, среднего чека, рентабельности и конверсии | Позволяет заменять алгоритмы расчёта KPI через единый интерфейс | Жёстко заданные методы расчёта |
| DTO | Обмен данными между FastAPI и сервисным слоем | Обеспечивает проверку структуры данных и стабильные контракты API | Передача словарей или ORM-объектов напрямую |
| Adapter | Интеграция с 1С, CSV/Excel и SMTP | Скрывает особенности внешних источников и сервисов | Прямые вызовы внешних API |
| Observer | Уведомление о новых данных и обновление дашборда | Позволяет реагировать на события без жёсткой связи компонентов | Периодический опрос |

## 2. Примеры реализации паттернов на Python

### 2.1. Repository

Repository предоставляет единый интерфейс для работы с данными о продажах и отделяет бизнес-логику от конкретного механизма хранения.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal


@dataclass
class SaleRecord:
    id: int | None
    month: str
    revenue: Decimal
    cost: Decimal


class SaleRepository(ABC):
    @abstractmethod
    def add(self, record: SaleRecord) -> None:
        pass

    @abstractmethod
    def get_by_period(self, period: str) -> list[SaleRecord]:
        pass


class InMemorySaleRepository(SaleRepository):
    def __init__(self):
        self.records: list[SaleRecord] = []

    def add(self, record: SaleRecord) -> None:
        self.records.append(record)

    def get_by_period(self, period: str) -> list[SaleRecord]:
        return [record for record in self.records if record.month == period]
```

В примере используется хранилище в памяти. При подключении PostgreSQL его можно заменить реализацией SqlSaleRepository, сохранив интерфейс SaleRepository.

### 2.2. Unit of Work

Unit of Work объединяет изменения в одну транзакцию. При успешном завершении операции изменения фиксируются, при ошибке выполняется откат.

```python
class UnitOfWork:
    def __init__(self, session_factory):
        self.session_factory = session_factory
        self.session = None

    def __enter__(self):
        self.session = self.session_factory()
        return self

    def commit(self):
        self.session.commit()

    def rollback(self):
        self.session.rollback()

    def __exit__(self, exc_type, exc_val, exc_tb):
        try:
            if exc_type is not None:
                self.rollback()
        finally:
            self.session.close()


# Пример использования с настроенной SQLAlchemy session_factory:
# with UnitOfWork(session_factory) as uow:
#     repository = SqlSaleRepository(uow.session)
#     repository.add(sale_record)
#     uow.commit()
```

В примере фиксация выполняется явно через commit(). Если при обработке возникает исключение, транзакция откатывается. Реализация предполагает использование сессии SQLAlchemy.

### 2.3. Strategy

Strategy позволяет выбирать алгоритм расчёта KPI без изменения кода сервиса, использующего стратегию.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class KpiData:
    revenue: float
    cost: float
    orders: int


class KpiStrategy(ABC):
    @abstractmethod
    def calculate(self, data: KpiData) -> float:
        pass


class ProfitabilityStrategy(KpiStrategy):
    def calculate(self, data: KpiData) -> float:
        if data.revenue == 0:
            return 0.0
        return (data.revenue - data.cost) / data.revenue * 100


class AverageCheckStrategy(KpiStrategy):
    def calculate(self, data: KpiData) -> float:
        if data.orders == 0:
            return 0.0
        return data.revenue / data.orders


class KpiCalculator:
    def __init__(self, strategy: KpiStrategy):
        self.strategy = strategy

    def calculate(self, data: KpiData) -> float:
        return self.strategy.calculate(data)


data = KpiData(revenue=100000, cost=70000, orders=200)

profitability = KpiCalculator(ProfitabilityStrategy())
average_check = KpiCalculator(AverageCheckStrategy())

print("Рентабельность:", profitability.calculate(data), "%")
print("Средний чек:", average_check.calculate(data))
```

Результат выполнения:

```text
Рентабельность: 30.0 %
Средний чек: 500.0
```

## 3. Связь паттернов с архитектурой

- Presentation: FastAPI-контроллеры используют DTO для приёма и возврата данных.
- Application: Service Layer организует импорт, расчёт KPI и формирование отчётов.
- Domain: Strategy определяет взаимозаменяемые алгоритмы расчёта показателей.
- Infrastructure: Repository, Unit of Work и Adapter обеспечивают доступ к данным, транзакции и интеграции.
- Factory используется при выборе генератора отчётов.
- Observer может использоваться для обработки событий обновления данных.

## 4. Вывод

Выбранные паттерны позволяют разделить ответственность между компонентами Business Metrics Analyzer, повысить тестируемость, упростить сопровождение и обеспечить возможность расширения приложения. Для учебного MVP их применение рассматривается как проектное архитектурное решение; реализация всех паттернов в рабочем приложении может выполняться постепенно.
  
