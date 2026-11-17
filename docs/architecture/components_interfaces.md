
# Интерфейсы и методы компонентов Business Metrics Analyzer

## 1. Назначение

Интерфейсы определяют способы взаимодействия компонентов приложения Business Metrics Analyzer. Они обеспечивают разделение ответственности между слоями архитектуры и позволяют изменять реализацию компонентов без изменения вызывающего кода.

## 2. Таблица интерфейсов и методов

| Компонент | Интерфейс | Ключевые методы |
|---|---|---|
| AuthService | IAuthService | authenticate(login, password) -> TokenDto; verify_token(token) -> UserDto |
| SalesImportService | ISalesImportService | import_csv(file) -> ImportResult; import_excel(file) -> ImportResult; validate(records) -> ValidationResult |
| KpiService | IKpiService | calculate_period(period) -> KpiResult; get_dashboard(period) -> DashboardDto |
| ReportService | IReportService | generate(report_type, period) -> ReportFile; export_excel(report_id) -> bytes; export_pdf(report_id) -> bytes |
| UserService | IUserService | get_user(user_id) -> UserDto; update_role(user_id, role) -> UserDto |
| SqlSaleRepository | ISaleRepository | add(record) -> None; get_by_period(period) -> list[SaleRecord] |
| SqlUserRepository | IUserRepository | get_by_id(user_id) -> User; get_by_login(login) -> User |
| UnitOfWork | IUnitOfWork | commit() -> None; rollback() -> None |
| JwtProvider | ITokenProvider | create_token(user_id, role) -> str; verify_token(token) -> dict |
| ExcelReportGenerator | IReportGenerator | generate(data) -> bytes |
| EmailNotifier | INotificationService | send_email(recipient, subject, message) -> None |

## 3. Взаимодействие компонентов

1. Контроллеры Presentation принимают HTTP-запросы и обращаются к интерфейсам сервисов Application.
2. Сервисы Application реализуют бизнес-сценарии: аутентификацию, импорт данных, расчёт KPI и формирование отчётов.
3. Компоненты Domain содержат модели данных, валидаторы и стратегии расчёта показателей.
4. Infrastructure реализует доступ к PostgreSQL, управление транзакциями, работу с JWT и интеграцию с внешними сервисами.
5. Использование интерфейсов уменьшает связанность компонентов и облегчает модульное тестирование.

## 4. Вывод

Выделение интерфейсов обеспечивает понятные контракты взаимодействия компонентов, упрощает сопровождение и позволяет расширять функциональность Business Metrics Analyzer без существенного изменения существующих модулей.
  
