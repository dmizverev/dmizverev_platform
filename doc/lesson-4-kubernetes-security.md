- Создан ServiceAccount bob. Создан ClusterRoleBinding adminbob, где для bob выдана роль admin.

- Создан ServiceAccount dave.

- Создан Namespace prometheus. В нем создан ServiceAccount carol.
Создана clusterrole с возможностью для Pod выполнять get list watch.
Эта роль выдана на namespace prometheus.

- Создан Namespace dev. В нем создан ServiceAccount jave с ролью admin и ServiceAccount ken
с ролью view.