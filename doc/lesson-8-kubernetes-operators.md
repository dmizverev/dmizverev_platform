- Создан CustomResource для заказа MySQL `kubernetes-operators\deploy\cr.yml

- Создан CustomResourceDefinition `kubernetes-operators\deploy\crd.yml`

- В CustomResourceDefinition добавлены validation
  
  В следствие этого из CustomResource удален `usless_data: "useless info"`, т.к. это не проходит валидацию.
  
- В CustomResourceDefinition добавлены required

  `required: ["image", "database", "password", "storage_size"]`
  
- CR и CRD применены в кластер Kubernetes
  ```bash
  kubectl apply -f deploy/crd.yml
  kubectl apply -f deploy/cr.yml
  ```
  Просмотр созданных объектов
  ```bash
  kubectl get crd
  kubectl get mysqls.otus.homework
  kubectl describe mysqls.otus.homework mysql-instance
  ```
- Создан MySQL оператор.

  Добавлены дополнительные функции 
  
  Запуск оператора:
  ```bash
  kopf run mysql-operator.py
  ```

- Вопрос: почему объект создался, хотя мы создали CR, до того, как
запустили контроллер?

  Оператор на kops реализует level triggered механизм опроса событий - опрос изменений во времени, а не последнего.
  
- Доработка оператора

  Доработана функция ``mysql_on_create``
  
  Создана функция `delete_object_make_backup`
  
  Добавлено создание PV и PVC для бэкапов.
  
  Добавлена функция ``delete_success_jobs``. 
  Хотя не понятно. У job существуют параметры их автоудаления.
  
  Добавлена функция `wait_until_job_end`
  
  Вызов созданных функций добавлен в `delete_object_make_backup`
  
  Добавлена генерация из шаблонов для restore-job
  
  Добавлено восстановление из бэкапов.
  
  Запуск контроллера:
  ```bash
  kopf run mysql-operator.py
  kubectl apply -f deploy/cr.yml
  ```
  
  Проверка:
  
  ```bash
  kubectl get pvc
  ```
  
  Заполнение созданной БД
  
  ```bash
  MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
  
  kubectl exec -it "${MYSQLPOD}" -- mysql -u root  -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database
  
  kubectl exec -it "${MYSQLPOD}" -- mysql -potuspassword  -e "INSERT INTO test ( id, name )VALUES ( null, 'some data' );" otus-database
  
  kubectl exec -it "${MYSQLPOD}" -- mysql -potuspassword -e "INSERT INTO test ( id, name )VALUES ( null, 'some data-2' );" otus-database
  ```
  
  Просмотр таблицы
  ```bash
  kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otusdatabase
  ```
  
- Создан Dockerfile для оператора: `kubernetes-operators\build\Dockerfile`

  Оператор собран и отправлен в docker hub `dmizverev/mysql-operator:0.1.0`
  
- Созданы файлы для установки оператора 
  deploy-operator.yml
  role.yml
  role-binding.yml
  service-account.yml

- Установка оператора

  ```bash
  kubectl apply -f deploy/
  
  ```
- Вывод команды `kubectl get jobs`

  ```
  NAME                         COMPLETIONS   DURATION   AGE
  backup-mysql-instance-job    1/1           1s         146s
  restore-mysql-instance-job   1/1           21s        87s
  ```
  
- Проверка созданной таблицы

  ```bash
  export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
  kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
  mysql: [Warning] Using a password on the command line interface can be insecure.
  +----+-------------+
  | id | name        |
  +----+-------------+
  |  1 | some data   |
  |  2 | some data-2 |
  +----+-------------+
  ```