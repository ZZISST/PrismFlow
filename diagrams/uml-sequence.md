# UML-диаграммы последовательности

## 1. Процесс ежедневного формирования прогноза

### Диаграмма

```mermaid
sequenceDiagram
    autonumber
    participant Scheduler as Scheduler<br/>(Airflow)
    participant FE as FeatureEngine
    participant FS as Feature Store<br/>(Feast)
    participant ForecastSvc as ForecastService<br/>(FastAPI)
    participant MR as ModelRegistry<br/>(MLflow)
    participant FDB as Forecast DB<br/>(PostgreSQL)
    participant OG as OrderGenerator
    participant ODB as Order DB<br/>(PostgreSQL)
    participant NS as Notification<br/>Service

    Note over Scheduler: Ежедневно в 02:00 MSK

    Scheduler->>FE: compute_features(date=today)
    activate FE
    FE->>FE: Загрузка данных из ClickHouse:<br/>продажи, остатки, погода, промо
    FE->>FE: Вычисление фичей:<br/>скользящие средние, тренды,<br/>сезонность, лаги
    FE->>FS: save_features(features_batch)
    activate FS
    FS-->>FE: OK (features saved)
    deactivate FS
    FE-->>Scheduler: OK (features computed)
    deactivate FE

    Scheduler->>ForecastSvc: run_prediction(date=today,<br/>horizon=7_days)
    activate ForecastSvc
    ForecastSvc->>MR: load_model(stage="production")
    activate MR
    MR-->>ForecastSvc: model artifact (LightGBM v2.3)
    deactivate MR

    ForecastSvc->>FS: get_features(stores=all,<br/>products=all)
    activate FS
    FS-->>ForecastSvc: feature_vectors (5M rows)
    deactivate FS

    ForecastSvc->>ForecastSvc: model.predict(features)<br/>→ 5M прогнозов<br/>(500 магазинов × 10K товаров)

    ForecastSvc->>FDB: save_forecasts(predictions)
    activate FDB
    FDB-->>ForecastSvc: OK (forecasts saved)
    deactivate FDB

    ForecastSvc->>OG: trigger_order_generation(date=today)
    activate OG
    ForecastSvc-->>Scheduler: OK (prediction complete)
    deactivate ForecastSvc

    OG->>FDB: read_forecasts(date=today)
    activate FDB
    FDB-->>OG: forecasts + current stock levels
    deactivate FDB

    OG->>OG: Расчёт заказов:<br/>predicted_qty − qty_on_hand<br/>+ safety_stock − qty_in_transit

    OG->>ODB: create_draft_orders(orders)
    activate ODB
    ODB-->>OG: OK (500 draft orders created)
    deactivate ODB

    OG->>NS: notify_managers(store_ids)
    activate NS
    NS->>NS: Формирование уведомлений<br/>(email + push)
    NS-->>OG: OK (notifications sent)
    deactivate NS
    deactivate OG

    Note over NS: Менеджеры получают<br/>уведомления к 06:00 MSK
```

### Описание процесса

Процесс ежедневного формирования прогноза запускается автоматически по расписанию (Airflow DAG) в 02:00 по московскому времени — в период минимальной нагрузки на системы.

**Этап 1: Вычисление фичей (шаги 1–5).** Scheduler запускает FeatureEngine, который загружает актуальные данные из ClickHouse (продажи за предыдущий день, текущие остатки, прогноз погоды, активные промоакции) и вычисляет ML-фичи: скользящие средние продаж за 7/14/30/90 дней, тренды, сезонные компоненты, лаговые переменные, промо-эффекты. Готовые фичи сохраняются в Feature Store (Feast) для обеспечения консистентности между обучением и инференсом.

**Этап 2: Инференс модели (шаги 6–11).** ForecastService загружает production-версию модели из MLflow Model Registry, получает фичи из Feature Store и выполняет предсказания для всех комбинаций товар × магазин × день на горизонт 7 дней. Это ~5 млн прогнозов (500 магазинов × 10 000 товаров). Результаты сохраняются в PostgreSQL.

**Этап 3: Генерация заявок (шаги 12–17).** OrderGenerator читает прогнозы и текущие остатки, рассчитывает оптимальный объём заказа по формуле: `заказ = прогноз − остаток + страховой запас − товар в пути`. Создаёт черновики заявок (статус `draft`) для каждого магазина и отправляет уведомления менеджерам. К 06:00 менеджеры получают готовые заявки для проверки.

---

## 2. Процесс корректировки заявки менеджером

### Диаграмма

```mermaid
sequenceDiagram
    autonumber
    participant Manager as Менеджер<br/>магазина
    participant WebUI as WebUI<br/>(React)
    participant API as DashboardAPI<br/>(FastAPI)
    participant ODB as Order DB<br/>(PostgreSQL)
    participant Audit as AuditLog

    Note over Manager: Утро, после получения<br/>уведомления о новой заявке

    Manager->>WebUI: Открыть дашборд →<br/>раздел "Заявки"
    activate WebUI

    WebUI->>API: GET /api/orders?store_id=42<br/>&status=draft
    activate API
    API->>ODB: SELECT orders, order_items<br/>WHERE store_id=42<br/>AND status='draft'
    activate ODB
    ODB-->>API: orders + items +<br/>forecast data
    deactivate ODB
    API-->>WebUI: OrderListResponse<br/>(заявки с прогнозами)
    deactivate API

    WebUI-->>Manager: Таблица заявок:<br/>товар | прогноз | заказ |<br/>остаток | рекомендация

    Manager->>Manager: Анализирует заявку,<br/>видит обоснование<br/>каждой позиции

    Manager->>WebUI: Корректирует позицию:<br/>"Хлеб белый" 120 → 150 шт<br/>Причина: "Ярмарка у магазина"

    WebUI->>API: PUT /api/orders/7854/items/312<br/>{qty: 150,<br/>adjustment_reason:<br/>"Ярмарка у магазина"}
    activate API

    API->>ODB: UPDATE order_items<br/>SET qty=150,<br/>qty_original=120,<br/>adjustment_reason=<br/>'Ярмарка у магазина'<br/>WHERE id=312
    activate ODB
    ODB-->>API: OK (1 row updated)
    deactivate ODB

    API->>Audit: INSERT audit_log<br/>(user, action, entity,<br/>old_value, new_value,<br/>reason, timestamp)
    activate Audit
    Audit-->>API: OK
    deactivate Audit

    API-->>WebUI: 200 OK<br/>(updated order item)
    deactivate API

    WebUI-->>Manager: Позиция обновлена ✓

    Manager->>WebUI: Нажимает<br/>"Подтвердить заявку"

    WebUI->>API: POST /api/orders/7854/approve
    activate API

    API->>ODB: UPDATE orders<br/>SET status='approved',<br/>approved_by=manager_id,<br/>approved_at=NOW()
    activate ODB
    ODB-->>API: OK
    deactivate ODB

    API->>Audit: INSERT audit_log<br/>(order approved)
    activate Audit
    Audit-->>API: OK
    deactivate Audit

    API-->>WebUI: 200 OK (order approved)
    deactivate API

    WebUI-->>Manager: Заявка подтверждена ✓<br/>Отправлена в РЦ
```

### Описание процесса

Процесс корректировки заявки начинается утром, когда менеджер магазина получает push-уведомление о сформированной автозаявке.

**Этап 1: Просмотр заявки (шаги 1–6).** Менеджер открывает веб-дашборд и переходит в раздел «Заявки». Система отображает список черновиков заявок для его магазина. Каждая позиция содержит: название товара, прогноз спроса модели, рекомендуемое количество заказа, текущий остаток и обоснование (почему модель рекомендует именно такое количество — например, «промоакция с понедельника, ожидаемый рост +40%»).

**Этап 2: Корректировка (шаги 7–13).** Менеджер анализирует заявку и может скорректировать отдельные позиции. В примере менеджер увеличивает заказ хлеба со 120 до 150 штук, указывая причину — ярмарка рядом с магазином (локальное знание, недоступное модели). Система сохраняет как новое, так и оригинальное количество, а также причину корректировки. Все изменения логируются в AuditLog для последующего анализа и обучения модели.

**Этап 3: Подтверждение (шаги 14–19).** После проверки всех позиций менеджер подтверждает заявку. Статус меняется с `draft` на `approved`, фиксируется время и идентификатор менеджера. Подтверждённая заявка автоматически отправляется в распределительный центр для комплектации. Корректировки менеджеров агрегируются и используются как сигнал для улучшения модели — если менеджеры систематически корректируют прогноз в одну сторону, это указывает на систематическую ошибку модели.
