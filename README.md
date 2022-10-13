# Ansible-роль `dockerized-service`

Ansible-роль для установки и запуска произвольного сервиса с помощью Docker Compose.

## Переменные Ansible-роли

#### Обязательные

| Переменная | Описание |
| --- | --- |  
| `compose` | Шаблон для формирования файла `docker-compose.yml`. Можно использовать подстановки Jinja2 |
| `dest` | Каталог, в котором формируется `docker-compose.yml` и куда копируются файлы и каталоги, указанные в `files` |
| `service_name` | Название сервиса. Может быть произвольным - используется только в сообщениях Ansible |

#### Необязательные

| Переменная           | Описание                                                                                           | Значение по-умолчанию |
|----------------------|----------------------------------------------------------------------------------------------------| --- |
| `down_before_up`     | Выполнять ли команду `docker-compose down` до команды `docker-compose up`                          | `false`
| `files`              | Список файлов/каталогов для копирования в `dest`                                                   | `[]`
| `service_port`       | Номер порта сервиса. Если не `0`, то после запуска сервиса происходит ожидание соединения с портом | `0`

#### Примеры использования

ActiveMQ:

```yaml
- hosts: activemq_server
  become: true
  roles:
    - role: dockerized-service
      vars:
        service_name: activemq
        service_port: 61616
        webconsole_port: 8161
        dest: /opt/{{ service_name }}
        compose: |
          version: "3.4"
          services:
            service:
              image: rmohr/activemq:5.15.9-alpine
              restart: always
              ports:
                - "{{ service_port }}:61616"
                - "{{ webconsole_port }}:8161"
              volumes:
                - data:/opt/activemq/data
          volumes:
            data:
```

MinIO:

```yaml
- hosts: minio_server
  become: true
  roles:
    - role: dockerized-service
      vars:
        service_name: minio
        service_port: 9000
        dest: /opt/{{ service_name }}
        compose: |
          version: "3.4"
          services:
            service:
              # MinIO использует Continuous Release Model, не ломающая
              # совместимость. Поэтому рекомендуется использовать тэг latest
              # (https://github.com/minio/minio/issues/2186)
              image: minio/minio:latest
              command: ['server', '/data']
              restart: always
              ports:
                - "{{ service_port }}:9000"
              environment:
                MINIO_ACCESS_KEY: '{{ MINIO.access_key }}'
                MINIO_SECRET_KEY: '{{ MINIO.secret_key }}'
              healthcheck:
                test: ['CMD', 'curl', '-f', 'http://localhost:9000/minio/health/live']
                interval: 30s
                timeout: 20s
                retries: 3
              volumes:
                - data:/data
          volumes:
            data:
```

Внутренний сервис подписывания документов:

```yaml
- hosts: signing_server
  become: true
  roles:
    - role: dockerized-service
      vars:
        service_name: signing-server
        service_port: 8182
        dest: /opt/{{ service_name }}
        files:
          - ../../backing-services/{{ service_name }}/application-std-{{ STAND }}.yml
          - ../../backing-services/{{ service_name }}/cryptopro-keys-std-{{ STAND }}
          - ../../backing-services/{{ service_name }}/docker-std-{{ STAND }}.env
          - ../../backing-services/{{ service_name }}/truststore
        down_before_up: yes
        compose: |
          version: "3.4"
          services:
            service:
              image: company/core/signing-server:1.0.2
              restart: always
              ports:
                - "{{ service_port }}:8080"
              env_file:
                - ./docker-std-{{ STAND }}.env
              volumes:
                - ./application-std-{{ STAND }}.yml:/opt/app/application.yml:ro
                - ./cryptopro-keys-std-{{ STAND }}:/keys:ro
                - ./truststore:/opt/app/truststore:ro
```

Разворачивание Node Exporter: 

```yaml
- role: dockerized-service
  vars:
    service_name: monitoring_node_exporter
    service_port: 59100
    dest: /opt/{{ service_name }}
    compose: |
      version: "3.4"
      services:
        service:
          image: prom/node-exporter:latest
          container_name: "{{ service_name }}"
          restart: always
          ports:
            - "{{ service_port }}:9100"
          volumes:
            - /etc/localtime:/etc/localtime:ro
            - /etc/timezone:/etc/timezone:ro
            - /proc:/host/proc:ro
            - /sys:/host/sys:ro
            - /:/rootfs:ro
          command:
            - '--path.procfs=/host/proc'
            - '--path.sysfs=/host/sys'
            - '--path.rootfs=/rootfs'
            - '--collector.filesystem.ignored-mount-points="^(/rootfs|/host|)/(sys|proc|dev|host|etc)($$|/)"'
            - '--collector.filesystem.ignored-fs-types="^(sys|proc|auto|cgroup|devpts|ns|au|fuse\.lxc|mqueue)(fs|)$$"'
```
