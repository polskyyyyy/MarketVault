# MarketVault
flowchart LR

    %% Источники данных
    subgraph Sources["Внешние источники данных"]
        API1["Yahoo Finance API"]
        API2["Alpha Vantage"]
        API3["Binance API"]
    end

    %% Потоковая интеграция
    Sources-->Kafka[(Kafka)]

    %% OLTP
    Kafka-->OLTP[(PostgreSQL (OLTP))]

    %% DWH загрузка и Data Vault
    OLTP-->Airbyte-->Staging[/"Staging Area"/]
    subgraph Greenplum["Greenplum DWH"]
        direction TB
        Staging-->RawVault["Raw Vault (Hubs, Links, Satellites)"]
        RawVault-->BusinessVault["Business Vault"]
    end

    %% Трансформации dbt
    Airflow-->dbt[("dbt")]
    dbt-->Staging
    dbt-->RawVault
    dbt-->BusinessVault

    %% Data Marts в ClickHouse
    BusinessVault-->ClickHouse[(ClickHouse Data Marts)]

    %% Аналитика и визуализация
    ClickHouse-->Superset[("Apache Superset")]

    %% ML и потоковые сигналы
    RawVault-- Исторические данные -->Spark["Spark (Training & Inference)"]
    Spark-->MLflow["MLflow (Model Registry)"]
    MLflow-->Spark
    Spark-- Предикты / сигналы -->BusinessVault
    Spark-- Предикты / сигналы -->ClickHouse

    %% Оркестрация
    Airflow[("Apache Airflow")]
    Airflow-->Airbyte
    Airflow-->Spark
    Airflow-->MLflow
    Airflow-->dbt

    %% Документация, Observability
    dbt-->dbtDocs["dbt docs"]
    Airflow-->Prometheus["Prometheus"]
    Prometheus-->Grafana["Grafana"]

    %% CI/CD и инфраструктура
    GitHubActions[("GitHub Actions CI/CD")]-->Terraform[("Terraform")]
    GitHubActions-->Ansible[("Ansible")]
    GitHubActions-->K8s["Kubernetes"]
    Terraform-->K8s
    Ansible-->K8s

    %% Подписи
    note right of GitHubActions: CI/CD для кода, моделей, инфры
    note bottom of Superset: Визуализация дашбордов

    %% Безопасность
    Airflow-->AirflowConnections["Airflow Connections (Secrets)"]
    subgraph GreenplumRoles["Роли и привилегии в Greenplum"]
        RawVault
        BusinessVault
    end
    GreenplumRoles:::auth

    classDef auth fill=#f0f0f0,stroke=#333,stroke-width=1px


Описание:

Sources: Внешние API, поставляющие данные.
Kafka: Потоковая шина для реального времени.
PostgreSQL (OLTP): Хранение транзакционных данных.
Airbyte: Загрузка данных в Staging Area Greenplum.
Greenplum DWH: Хранение и трансформация данных по методологии Data Vault (Raw Vault → Business Vault).
dbt: Управление SQL-моделями и трансформациями Data Vault, тесты данных.
ClickHouse: Хранение Data Marts для быстрой аналитики.
Superset: Визуализация и дашборды.
Spark и MLflow: ML-процессы, обучение, предикты, сигналы.
Airflow: Оркестрация всех процессов (интеграции, трансформаций, ML-тасков).
Prometheus + Grafana: Мониторинг и метрики.
GitHub Actions, Terraform, Ansible, Kubernetes: CI/CD и инфраструктура как код.