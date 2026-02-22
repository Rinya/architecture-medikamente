## Что улучшать

#### 3.1 Каталог чувствит. данных и методы защиты

| Идентификатор | Класс данных | Подтип | Метод защиты |
|---------------|-------------|--------|--------------|
| D-01 | Персональные | ФИО | Шифрование AES-256 (at rest), TLS 1.3 (in transit), обфускация при выгрузке в тест |
| D-02 | Персональные | тел./e-mail | То же + маскировка (***@yandex.ru) в логах |
| D-03 | PHI | Диагноз, жалобы | Tokenization + database-level encryption, согласие, role-based redaction |
| D-04 | PHI-анализы | PDF-файлы | File-level AES, vault-ключ, secure S3, динамическое obfuscation при preview |
| D-05 | Финансы | Чек, маскированная карта | Токенизировать PAN, оставлять только последние 4, шифрование AES-GCM |
| D-06 | Секреты сотрудников | З/п | Column-level encryption + ABAC |
| D-07 | Контрагент-PД | Поставщик-контакт | Уничтожать после завершения сделки, обезличивать для отчётности |

---

#### 3.2 Механизм тегирования (Data Tagging)

**Реализация (пример):**
- Приём ресепшена → заполняется форма, в которой поля помечены служебным атрибутом:  
  `sensitivity=high|medium|low`  
- При сохранении в БД добавляется столбец `tag_mask (bit)`  
- При выдаче API поле с `high` шифруется или вырезается для ролей без `ROLE_SENSITIVE`.

**Инструменты:**
- Apache NiFi: атрибут `sensitivity.level` на flow-file, router «Encrypt-if-high».  
- Postgres + pgcrypto: `CASE WHEN tag='high' THEN pgp_sym_encrypt(value, key) ELSE value END`.  
- Spring-AOP (для будущих порталов): аннотация `@Sensitive(high)` → автоматическая маршрутизация в SecureService.

---

#### 3.3 Состав «чек-лист» средств защиты

| Цель | Технология/процесс |
|------|-------------------|
| **Данные в покое** | DB-TDE (Postgres), LUKS-диск, Azure/AWS S3 SSE-KMS |
| **Транспорт** | Внутри NiFi – TLS-контекст. API – mTLS + JWT (OAuth). Очереди – Kafka-SSL |
| **Аутентификация** | SAML/SSO, Azure AD, PKI-карты для врачей |
| **Авторизация** | Keycloak/OPA (ABAC), роли `DOCTOR_REGION_A`, `RECEPTION_OWN_PATIENT_ONLY` |
| **Маскирование** | Dynamic Data Masking (Postgres plugin), FPE-Format-Preserving |
| **Удаление (RTBF)** | Service «ForgetPatient» – вычищает по ID, шедулер «stale-data» |
| **Логирование** | ELK → Filebeat → secured index `audit-*`, immutable, 1 год |
| **Мониторинг** | Victoria-Metrics-аномалия по download > N файлов/час, Alertmanager→Telegram+SIEM |
| **Backup** | pg_basebackup + pg_dump в S3 (SSE-KMS), 30 дней hot, 1 год cold |
