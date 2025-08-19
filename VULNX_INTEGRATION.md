# 🚀 Интеграция vulnx в пенетрационный фреймворк

Интеграция современного инструмента [vulnx (cvemap)](https://github.com/projectdiscovery/cvemap) от ProjectDiscovery для автоматического поиска эксплойтов по найденным уязвимостям.

## 📋 Архитектура

### 🔄 Event-driven модель
- **Автоматический мониторинг** новых CVE в базе данных
- **Триггерная обработка** при появлении уязвимостей с CVE ID
- **Кэширование результатов** для избежания повторных запросов к API
- **Приоритизация эксплойтов** по типу, языку программирования и CVSS

### 🗃️ Расширенная схема БД
```sql
-- Таблица найденных эксплойтов
exploits (
    id, vulnerability_id, cve_id, exploit_type, source,
    title, description, url, file_path, language,
    severity_score, is_working, metadata, created_at
)

-- Кэш запросов к vulnx API
cve_cache (
    id, cve_id, vulnx_response, exploits_found,
    last_checked, is_stale
)

-- Отслеживание статуса обработки
cve_processing (
    id, vulnerability_id, cve_id, status,
    vulnx_checked, nuclei_checked, exploits_downloaded,
    last_processed, error_message
)
```

## 🛠️ Компоненты

### 1. **VulnXProcessor** (`scanner/vulnx_processor.py`)
Основной модуль для работы с vulnx:
- Автоматическая установка vulnx через `go install`
- Извлечение CVE ID из текста уязвимостей
- Запросы к vulnx API с кэшированием
- Парсинг и нормализация данных эксплойтов
- Приоритизация по severity и типу

### 2. **CVEMonitor** (`scanner/cve_monitor.py`)  
Event-driven обработчик:
- Мониторинг новых уязвимостей в реальном времени
- Автоматическая обработка CVE через vulnx
- Уведомления о найденных эксплойтах
- Повтор failed обработок
- Управление устаревшим кэшем

### 3. **CLI интеграция** (`cli.py`)
Новая команда `exploits` с подкомандами:
- `search` - поиск эксплойтов для pending уязвимостей
- `monitor` - запуск мониторинга новых CVE
- `status` - статус обработки CVE
- `report` - отчёт по найденным эксплойтам

## 📚 Использование

### Базовые команды

```bash
# Поиск эксплойтов для всех pending уязвимостей
poetry run python cli.py exploits search

# Поиск с ограничением количества
poetry run python cli.py exploits search --limit 20

# Запуск автоматического мониторинга (каждые 2 минуты)
poetry run python cli.py exploits monitor --interval 120

# Мониторинг в фоне
poetry run python cli.py exploits monitor --daemon --interval 300

# Статус обработки CVE
poetry run python cli.py exploits status

# Отчёт по найденным эксплойтам
poetry run python cli.py exploits report

# JSON отчёт для автоматизации
poetry run python cli.py exploits report --format json
```

### Workflow

#### 1. **Первичное сканирование**
```bash
# Обычное сканирование для поиска уязвимостей
poetry run python cli.py full-scan https://example.com --db scan_results.db
```

#### 2. **Поиск эксплойтов**
```bash
# Поиск эксплойтов для найденных CVE
poetry run python cli.py exploits search --db scan_results.db

# Результат:
# [INFO] Поиск эксплойтов для уязвимостей...
# [SUCCESS] Обработано уязвимостей: 15
# [SUCCESS] Найдено эксплойтов: 42
```

#### 3. **Мониторинг новых CVE**
```bash
# Запуск автоматического мониторинга
poetry run python cli.py exploits monitor --interval 120 --daemon

# Вывод:
# [INFO] Запуск мониторинга CVE (интервал: 120s)
# [INFO] Мониторинг запущен в фоне
# [STATUS] Обработано CVE: 25, Найдено эксплойтов: 67
```

#### 4. **Анализ результатов**
```bash
# Статус обработки
poetry run python cli.py exploits status

# Результат:
# === Статус CVE обработки ===
# Мониторинг активен: ✅
# Последняя проверка: 2025-01-19T10:15:30
# 
# 📊 Статистика обработки:
#   ✅ completed: 23
#   ❌ failed: 2
#   ⏳ processing: 1
# 
# 🎯 Статистика эксплойтов:
#   💥 Всего эксплойтов: 67
#   🔍 Уникальных CVE: 23
#   🎯 Уязвимых ресурсов: 12
```

#### 5. **Отчёт по эксплойтам**
```bash
# Детальный отчёт
poetry run python cli.py exploits report

# Результат:
# === 📋 Отчёт по найденным эксплойтам ===
# 
# 📊 Статистика по типам эксплойтов:
#   🔸 poc (github, python): 23 эксплойтов
#   🔸 exploit (exploitdb, c): 18 эксплойтов
#   🔸 nuclei_template (nuclei, yaml): 12 эксплойтов
# 
# 🎯 Топ CVE по количеству эксплойтов:
#   🔴 CVE-2024-1234: 8 эксплойтов (severity: 9.1)
#   🟠 CVE-2024-5678: 5 эксплойтов (severity: 7.8)
```

## ⚙️ Конфигурация

### Установка зависимостей
```bash
# Основные зависимости уже в pyproject.toml
poetry install

# vulnx устанавливается автоматически при первом запуске
# Или вручную:
go install -v github.com/projectdiscovery/cvemap/cmd/cvemap@latest
```

### Переменные окружения
```bash
# API ключ ProjectDiscovery для увеличения rate limits (опционально)
export PDCP_API_KEY="your_api_key_here"

# Настройка vulnx auth (если нужно)
vulnx auth --api-key "$PDCP_API_KEY"
```

### Настройки кэширования
```python
# В VulnXProcessor можно настроить:
processor = VulnXProcessor(
    db_path="scan_results.db",
    cache_days=7  # Срок жизни кэша
)
```

## 🎯 Типы эксплойтов

### **PoC (Proof of Concept)**
- **Источник**: GitHub репозитории
- **Описание**: Демонстрационные скрипты
- **Языки**: Python, JavaScript, Bash, PHP
- **Приоритет**: Средний-высокий

### **Эксплойты**
- **Источник**: ExploitDB, Packet Storm
- **Описание**: Готовые эксплойты
- **Языки**: C, Python, Ruby, Perl
- **Приоритет**: Высокий

### **Nuclei Templates**
- **Источник**: nuclei-templates репозиторий
- **Описание**: YAML шаблоны для проверки
- **Языки**: YAML
- **Приоритет**: Высокий (автоматизированная проверка)

## 🔧 Расширение функциональности

### Добавление новых источников эксплойтов

```python
# В VulnXProcessor._parse_exploit_item()
# Добавьте новый источник:

if 'packetstorm' in item and item['packetstorm']:
    for ps_item in item['packetstorm']:
        exploit = base_info.copy()
        exploit.update({
            'exploit_type': 'exploit',
            'source': 'packetstorm',
            'url': ps_item.get('url'),
            'title': ps_item.get('title'),
            'language': self._detect_language_from_title(ps_item.get('title', ''))
        })
        exploits.append(exploit)
```

### Интеграция с уведомлениями

```python
# В CVEMonitor._notify_exploits_found()
def _notify_exploits_found(self, vulnerability, result):
    # Slack webhook
    webhook_url = "https://hooks.slack.com/services/..."
    message = {
        "text": f"🚨 Найдены эксплойты для {vulnerability['resource']}",
        "attachments": [{
            "color": "danger",
            "fields": [
                {"title": "CVE", "value": cve_id, "short": True}
                for cve_result in result['processed_cves']
                for cve_id in [cve_result['cve_id']]
                if cve_result['exploits_count'] > 0
            ]
        }]
    }
    requests.post(webhook_url, json=message)
```

### Автоматическое тестирование эксплойтов

```python
# Расширение для автоматического тестирования PoC
class ExploitTester:
    def test_poc(self, exploit_data):
        if exploit_data['language'] == 'python' and exploit_data['source'] == 'github':
            # Скачиваем и тестируем Python PoC в sandbox
            return self._test_python_poc(exploit_data['url'])
        return None
```

## 📊 Мониторинг производительности

### Метрики vulnx
- **Rate limits**: без API ключа - 10 запросов/минуту
- **Кэширование**: 7 дней по умолчанию
- **Timeout**: 30 секунд на запрос
- **Retry**: автоматический повтор failed через 1 час

### Оптимизация
```python
# Настройка параллельной обработки
async def process_multiple_cves(cve_list):
    tasks = [
        process_cve_async(cve_id) 
        for cve_id in cve_list
    ]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

## 🔗 Интеграция с Nuclei

После нахождения nuclei template через vulnx можно автоматически запустить проверку:

```python
def auto_nuclei_check(exploit_data):
    if exploit_data['exploit_type'] == 'nuclei_template':
        template_url = exploit_data['url']
        target = exploit_data['target']
        
        # Скачиваем template
        template_path = download_nuclei_template(template_url)
        
        # Запускаем nuclei с конкретным шаблоном
        cmd = ['nuclei', '-t', template_path, '-u', target, '-jsonl']
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        return parse_nuclei_output(result.stdout)
```

## 🚀 Roadmap

### Планируемые улучшения
- [ ] **Автоматическое скачивание эксплойтов** в локальную папку
- [ ] **Sandbox тестирование PoC** в изолированной среде  
- [ ] **Интеграция с Nuclei** для автоматической проверки найденных шаблонов
- [ ] **Webhook уведомления** в Slack/Discord/Telegram
- [ ] **ML классификация эксплойтов** по релевантности
- [ ] **Автоматическое создание Metasploit модулей** из PoC
- [ ] **Интеграция с SIEM системами** (Splunk, ELK)

### Возможные источники данных
- [ ] **GitHub Code Search** для поиска PoC в коде
- [ ] **Twitter/X Search** для mention CVE ID
- [ ] **Reddit Security** сабреддиты
- [ ] **CVE Details** для дополнительной информации
- [ ] **MITRE ATT&CK** для mapping техник

---

*Интеграция разработана для максимальной автоматизации поиска и анализа эксплойтов в пенетрационном тестировании.*
