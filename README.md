# Kubernetes + Istio: готовая лаборатория для управления микросервисами 🚀

Практический проект для тех, кто хочет **на реальном примере разобраться, как Kubernetes и Istio помогают управлять микросервисами, безопасностью, трафиком и отказоустойчивостью**.

Вместо изучения отдельных YAML-конфигураций здесь можно запустить небольшой микросервисный стенд и увидеть, что происходит с приложением при изменении маршрутизации, включении mTLS, ограничении запросов и распределении трафика между версиями сервиса.

## 🎯 Что вы получите от проекта

После запуска проекта вы сможете на практике:

* понять, как Istio управляет взаимодействием микросервисов;
* увидеть работу **Service Mesh** поверх Kubernetes;
* настроить **Canary Deployment** и распределять трафик между версиями приложения;
* безопасно передавать данные между сервисами с помощью **mTLS**;
* ограничивать количество запросов через **Rate Limiting**;
* настроить **Retries** и **Circuit Breaker** для повышения устойчивости;
* автоматически масштабировать приложение с помощью **HPA**;
* визуально исследовать трафик через **Kiali**;
* находить проблемные места и отслеживать запросы с помощью **Jaeger**;
* разобраться, какую задачу решает каждая конфигурация Istio и Kubernetes.

Проект особенно полезен как **практическая лаборатория для изучения Kubernetes, Istio и принципов эксплуатации микросервисов**.

---

## 🧩 Что можно изучить на проекте

Проект состоит из двух Go-сервисов:

```text
                    ┌──────────────────┐
                    │   Istio Gateway  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Frontend Service │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Istio / Envoy  │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    ▼                  ▼
             Backend v1          Backend v2
                    │                  │
                    └────────┬─────────┘
                             ▼
                    Kubernetes Cluster
```

### Frontend

Принимает внешний HTTP-трафик и обращается к Backend.

### Backend v1 / v2

Две версии одного сервиса позволяют наглядно продемонстрировать **Canary Deployment** и постепенный перевод пользователей на новую версию.

Все взаимодействия между сервисами проходят через Envoy Sidecar, управляемый Istio.

---

# 🛠 Быстрый запуск

## 1. Подготовить Minikube

Проект рассчитан на запуск локального Kubernetes-кластера через Minikube.

```bash
minikube start --cpus=4 --memory=8192 --disk-size=20g --driver=docker

kubectl cluster-info
```

---

## 2. Установить Istio

Скачать Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Перейти в каталог Istio:

```bash
cd istio-1.29.1
export PATH=$PWD/bin:$PATH
```

Установить Istio:

```bash
istioctl install --set profile=demo -y
```

Проверить установку:

```bash
istioctl verify-install
```

---

# 📦 3. Развернуть приложение

Создать namespace:

```bash
kubectl create namespace mrm-project
```

Включить автоматическое добавление Istio Sidecar:

```bash
kubectl label namespace mrm-project istio-injection=enabled
```

## Собрать Docker-образы внутри Minikube

```bash
eval $(minikube docker-env)
```

Собрать Backend:

```bash
docker build -t mrm-backend:v1 ./services/backend
docker build -t mrm-backend:v2 ./services/backend
```

Собрать Frontend:

```bash
docker build -t mrm-frontend:v1 ./services/frontend
```

## Применить Kubernetes-манифесты

```bash
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

Проверить состояние Pod:

```bash
kubectl get pods -n mrm-project
```

---

# 🌐 4. Управление трафиком через Istio

## Istio Gateway

```bash
kubectl apply -f istio-configs/gateway.yaml
kubectl apply -f istio-configs/frontend-vs.yaml
```

Получить адрес приложения:

```bash
minikube service istio-ingressgateway -n istio-system --url
```

Теперь внешний трафик проходит через Istio Gateway.

---

# 🐤 5. Canary Deployment 80/20

Одна из главных практических демонстраций проекта — постепенное переключение пользователей между версиями Backend.

```bash
kubectl apply -f istio-configs/backend-split.yaml
```

Например:

```text
             Incoming Traffic
                    │
                    ▼
             Istio VirtualService
                /           \
             80%             20%
              │               │
              ▼               ▼
         Backend v1       Backend v2
```

Это позволяет увидеть на практике, как можно выпустить новую версию сервиса, не переводя на неё сразу весь трафик.

Проверить распределение запросов:

```bash
while true; do
  curl -s $(minikube service istio-ingressgateway -n istio-system --url | head -n 1) | grep "Response"
  sleep 0.5
done
```

---

# 🛡 6. Отказоустойчивость и защита

## Rate Limiting

Ограничение количества запросов позволяет контролировать нагрузку на сервис.

```bash
kubectl apply -f istio-configs/rate-limit.yaml
```

Это полезный пример того, как часть защитной логики можно вынести из приложения на уровень инфраструктуры.

---

## HPA

Автоматическое масштабирование приложения:

```bash
kubectl apply -f k8s/hpa.yaml
```

Kubernetes сможет изменять количество Pod в зависимости от нагрузки.

---

## Retries

Если временный сетевой сбой приводит к неудачному запросу, Istio может автоматически выполнить повторную попытку.

Это позволяет уменьшить влияние кратковременных сбоев на пользователя.

---

## Circuit Breaker

Circuit Breaker помогает ограничить воздействие проблемного сервиса на остальные компоненты системы.

Если зависимость начинает работать нестабильно или перегружена, ограничения позволяют предотвратить дальнейшее распространение проблемы.

---

# 🔒 7. mTLS

Включить строгий режим шифрования между сервисами:

```bash
kubectl apply -f istio-configs/security.yaml
```

В результате коммуникация между сервисами защищается механизмом Istio mTLS.

На этом примере можно изучить:

* шифрование service-to-service трафика;
* автоматическое управление сертификатами;
* принцип Zero Trust внутри Service Mesh;
* отличие обычного сетевого взаимодействия от защищённого взаимодействия через Istio.

---

# 👁 8. Observability

Микросервисная архитектура создаёт проблему: один пользовательский запрос может пройти через несколько сервисов.

Поэтому проект показывает не только работу сервисов, но и инструменты для понимания происходящего внутри системы.

## Kiali

Установить Kiali:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.29/samples/addons/kiali.yaml
```

Открыть:

```bash
istioctl dashboard kiali
```

Kiali позволяет визуально посмотреть Service Mesh и увидеть связи между сервисами.

---

## Jaeger

Установить Jaeger:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.29/samples/addons/jaeger.yaml
```

Открыть:

```bash
istioctl dashboard jaeger
```

Jaeger позволяет исследовать трассировку запросов и определить, на каком этапе распределённого вызова возникает задержка.

---

# 🧠 Какие реальные задачи демонстрирует проект

| Задача                     | Что показывает проект                     |
| -------------------------- | ----------------------------------------- |
| Управление трафиком        | Istio Gateway и VirtualService            |
| Canary-релизы              | Распределение трафика между v1 и v2       |
| Service Discovery          | Взаимодействие сервисов внутри Kubernetes |
| Безопасность               | mTLS между сервисами                      |
| Защита от нагрузки         | Rate Limiting                             |
| Восстановление после сбоев | Retries                                   |
| Изоляция проблем           | Circuit Breaker                           |
| Масштабирование            | Kubernetes HPA                            |
| Визуализация               | Kiali                                     |
| Distributed Tracing        | Jaeger                                    |

---

# 📁 Структура проекта

```text
.
├── istio-configs/
│   ├── gateway.yaml
│   ├── frontend-vs.yaml
│   ├── backend-split.yaml
│   ├── rate-limit.yaml
│   └── security.yaml
│
├── k8s/
│   ├── backend.yaml
│   ├── frontend.yaml
│   └── hpa.yaml
│
├── services/
│   ├── backend/
│   └── frontend/
│
├── istio-1.29.1/
├── go.mod
└── README.md
```

---

# 💡 Для кого этот проект

Проект может быть полезен:

* разработчикам Go, которые хотят понять эксплуатацию микросервисов в Kubernetes;
* DevOps и SRE-инженерам;
* тем, кто изучает Istio и Service Mesh;
* разработчикам, изучающим Canary Deployment;
* специалистам, которым нужно разобраться с mTLS, observability и resilience;
* тем, кто хочет иметь локальную лабораторию для экспериментов с Kubernetes и Istio.

---

# 🚀 Что можно попробовать самостоятельно

После запуска проекта можно менять конфигурации и наблюдать результат:

1. Изменить соотношение трафика v1/v2.
2. Добавить третью версию Backend.
3. Изменить правила Rate Limiting.
4. Включить или изменить Retries.
5. Настроить Circuit Breaker.
6. Изменить правила mTLS.
7. Увеличить нагрузку и посмотреть работу HPA.
8. Найти запрос в Jaeger и проследить его путь.
9. Посмотреть изменение графа сервисов в Kiali.

Таким образом, репозиторий можно использовать не только для просмотра исходников, но и как **готовый стенд для экспериментов с Kubernetes и Istio**.

---

# 📌 Итог

Этот проект показывает полный путь от обычных Go-микросервисов до управляемой инфраструктуры Kubernetes:

**микросервисы → Kubernetes → Istio → управление трафиком → безопасность → отказоустойчивость → масштабирование → наблюдаемость.**

Главная ценность проекта — возможность **не просто прочитать о возможностях Istio, а самостоятельно запустить их и увидеть результат на работающем микросервисном приложении**.
