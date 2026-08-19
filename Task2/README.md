# Задание 2. Динамическое масштабирование контейнеров

## Часть 1. Масштабирование по памяти

Файлы:
- [deployment.yaml](deployment.yaml) - Deployment тестового приложения scaletestapp (1 реплика, memory limit/request 30Mi).
- [service.yaml](service.yaml) - Service (NodePort) для доступа к приложению, с аннотациями `prometheus.io/scrape` для части 2.
- [hpa-memory.yaml](hpa-memory.yaml) - HPA по утилизации памяти (target 80%, min 1 / max 10 реплик).

Нагрузка сгенерирована locust (`locust --headless -u 400 -r 20`) через `kubectl port-forward` на Service. Под нагрузкой утилизация памяти пода выросла с ~20% в состоянии покоя до 92%, после чего HPA увеличил число реплик с 1 до 2 (событие `SuccessfulRescale: New size: 2; reason: memory resource utilization above target`).

Заметное поведение: у HPA есть встроенный tolerance 10% - при пересечении target 80% реального порога масштабирование срабатывает не сразу, а примерно с 88%. При этом лимит памяти в 30Mi настолько мал, что приложение периодически падает по OOMKilled раньше, чем накопится достаточно данных для срабатывания HPA - это иллюстрирует важность подбора адекватных лимитов ресурсов при настройке автомасштабирования.

Логи и скриншоты:
- [logs/hpa-memory-scale-events.txt](logs/hpa-memory-scale-events.txt) - события HPA и динамика утилизации памяти по времени.
- <img src="/images/tasks/Task2/dashboard-deployment-memory-scale.png" alt="Kubernetes Dashboard: Deployment scaletestapp, 2/2 pods"/>
- <img src="/images/tasks/Task2/dashboard-pods-memory-scale.png" alt="Kubernetes Dashboard: Pods scaletestapp после масштабирования по памяти"/>

## Часть 2. Масштабирование по RPS

Дополнительно установлены:
- Prometheus (Helm chart `prometheus-community/prometheus`) с job'ом `kubernetes-pods`, который по аннотациям пода (`prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path`) собирает метрику `http_requests_total`, проставляя лейбл `pod`.
- Prometheus Adapter (Helm chart `prometheus-community/prometheus-adapter`), который транслирует `sum(rate(http_requests_total{...}[2m])) by (pod)` в custom-метрику `http_requests_per_second`, доступную через Custom Metrics API (`pods/http_requests_per_second`).
- [hpa-rps.yaml](hpa-rps.yaml) - обновлённый HPA типа `Pods`, масштабирующий по `http_requests_per_second` c целевым средним значением 20 rps на под (min 1 / max 10 реплик).

Проверка сбора метрик в Prometheus:
- <img src="/images/tasks/Task2/prometheus-targets.png" alt="Prometheus Targets: job kubernetes-pods UP"/>
- <img src="/images/tasks/Task2/prometheus-graph-http-requests-total.png" alt="Prometheus Graph: http_requests_total по подам"/>

Нагрузочный тест (locust, 200 пользователей) показал последовательное масштабирование по мере роста RPS:
- 0/20 rps -> 2 реплики (стартовое состояние после части 1);
- ~209 rps -> масштабирование до 4 реплик;
- ~299 rps -> масштабирование до 8 реплик;
- ~344-469 rps -> масштабирование до 10 реплик (упёрлись в `maxReplicas`, статус `ScalingLimited: TooManyReplicas`).

Логи и скриншот:
- [logs/hpa-rps-scale-events.txt](logs/hpa-rps-scale-events.txt) - события HPA и динамика RPS-метрики по времени.
- <img src="/images/tasks/Task2/dashboard-pods-rps-scale.png" alt="Kubernetes Dashboard: Pods scaletestapp после масштабирования по RPS"/>

## Особенности реализации

Образ `scaletestapp` публикуется только под архитектуру `amd64`, а локальный кластер Minikube в этом окружении работает на арм-ноде (Apple Silicon). Kubelet отказывается тянуть образ напрямую из-за отсутствия подходящего manifest для `arm64` в manifest list, поэтому образ был предварительно загружен в контейнер-рантайм ноды через `minikube image load` (с использованием эмуляции amd64 в Docker Desktop), а в Deployment выставлен `imagePullPolicy: IfNotPresent`.
