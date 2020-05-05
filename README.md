# dmizverev_platform
Dmitriy Zverev Platform repository
dmizverev@yandex.ru

# Урок №2

> Разберитесь почему все pod в namespace kube-system
восстановились после удаления. Укажите причину в описании PR

Предположу, что: 
- kube-apiserver - это static Pod. За его восстановлением следит непосредственно kubelet.
См. https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/
- За восстановление cor-edns, kube-dns отвечает AddonManager, он реагирует на restartpolicy.

> Для выполнения домашней работы необходимо создать
> Dockerfile, в котором будет описан образ, запускающий web-сервер на порту 8000

Создан docker image с python web-server `http.server`. 
Docker image размещен в registry `docker.io/dmizverev/otus-frontend:1.0.0`.

> Напишем манифест web-pod.yaml для создания pod web c
> меткой app со значением web, содержащего один контейнер с
> названием web.

Выполнено.

> Добавим в наш pod init-container, генерирующий страницу
> index.html.

Выполнено.

> Для того, чтобы файлы, созданные в init контейнере, были
> доступны основному контейнеру в pod нам понадобится
> использовать volume типа emptyDir

Выполнено.

> Склонируйте и соберите собственный образ для
frontend (используйте готовый Dockerfile)
Поместите собранный образ на Docker Hub

Выполнено

> генерация манифестов средствами kubectl

Выполнено. Файл: frontend-pod.yaml

> Выясните причину, по которой pod frontend находится в статусе
Error

Не заданы переменные окружения

> Создайте новый манифест frontend-pod-healthy.yaml. При его
применении ошибка должна исчезнуть

Выполнено.