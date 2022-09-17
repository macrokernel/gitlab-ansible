# Ansible playbook for GitLab deployment in Docker Compose with Yandex Mail, Yandex Passport authentication and Yandex Cloud S3 backups

Плейбук Ansible для развёртывания GitLab и GitLab runner в Docker Compose с отправкой email через Yandex Mail, аутентификацией пользователей в Yandex Passport и резервным копированием в хранилище S3 Yandex Cloud.


## Подготовка

1. На целевых узлах разрешить беспарольное использование команды **sudo** для пользователя **ansible_user**.
2. Сгенерировать конфигурационные файлы (**ansible/inventory.ini** и **ansible/group_vars/all.yml**), выполнив команду:
```shell
ansible-playbook 0_generate_configs.yml
```
 - Отредактировать **ansible/inventory.ini**, указав адреса IP или DNS и имена пользователей целевых узлов
 - Отредактировать **ansible/group_vars/all.yml**, указав параметры развертывания


## Новое развёртывание

1. Выполнить последовательность команд:
```shell
cd ./ansible
ansible-playbook -i inventory.ini 2_configure_ufw.yml
ansible-playbook -i inventory.ini 3_install_docker.yml
```
2. Проверить параматры GitLab в файле шаблона **ansible/roles/install_gitlab/templates/gitlab.rb.j2**. Параметры GitLab, определённые в означенном файле шаблона приведены в Приложении 1.
3. Проверить сценарий настройки GitLab после установки в файле шаблона **ansible/roles/install_gitlab/templates/gitlab-post-reconfigure.sh.j2**. В частности, команду **ApplicationSetting.last.update(domain_allowlist: ["{{ gitlab_smtp_domain }}"])**, устанавливающую перечень доменов email, для которых разрешён вход в GitLab.
4. Для того, чтобы GitLab мог отправлять email через Yandex, необходимо войти в веб-интерфейс почты Yandex с учётной записью почты GitLab и принять условия EULA.  
Указать пароль учётной записи электронной почты GitLab в параметре **gitlab_smtp_password** файла **ansible/group_vars/all.yml**.
5. Для того, чтобы GitLab мог загружать бэкапы в хранилище S3 в облаке Yandex, необходимо войти в панель управления облака Yandex с учётной записью администратора и создать сервисный аккаунт для работы с бэкапами GitLab:  
  **Имя: gitlab-backup**  
  **Роль: storage.editor**  
Затем создать статический ключ доступа для данного сервисного аккаунта.  
После чего указать полученный идентификатор ключа и ключ в параметрах файла **ansible/group_vars/all.yml**:
```yaml
s3_access_key: "AKIAKIAKI"
s3_secret_key: "secret123"
```
6. Для интеграции GitLab с сервисом аутентификации Yandex Passport необходимо определить следующие параматры в файле **ansible/group_vars/all.yml**:
```yaml
oauth_app_id: "EXAMPLEAPPID"
oauth_app_secret: "EXAMPLESECRET"
```
7. Для обеспечения возможности регистрации GitLab runner необходимо сгенерировать строку, состоящую из случайного набора 20 букв различного регистра и цифр, и указать её в качестве значения параметра **gitlab_runner_initial_registration_token** в файле **ansible/group_vars/all.yml**. Для генерации случуайной строки можно воспользоваться командой
```shell
tr -cd '[:alnum:]' < /dev/urandom | fold -20 | head -n1
```
8. Для установки GitLab выполнить команду
```shell
ansible-playbook -i inventory.ini 4_install_gitlab.yml
```
9. Временный пароль администратора для первого входа в веб-интерфейс GitLab содержится в файле **/opt/gitlab/etc/initial_root_password** на сервере GitLab, пользователь - **root**. Данный пароль действует в течение 24 часов после установки GitLab.
После установки необходимо назначить как минимум одного пользователя GitLab администратором. Для этого следует войти в интерфейс GitLab посредством Yandex Passport с учётной записью пользователя, которому требуется предоставить полномочия администратора, для того, чтобы означенный пользователь был добавлен в базу данных GitLab, затем выйти из интерфейса GitLab. После чего войти в интерфейс GitLab с учётной записью **root**, перейти в раздел https://gitlab.example.com/admin/users и изменить уровень доступа пользователя на **Administrator**. Далее, выйти из интерфейса GitLab и снова войти с учётной записью пользователя-администратора.  
Затем следует запретить самостоятельную регистрацию (sign-up) пользователей и вход в веб-интерфейс GitLab по паролю (password authentication) на странице http://gitlab.example.com/admin/application_settings/general в разделах **Sign-up restrictions** и **Sign-in restrictions**, соответственно.
10. Установку GitLab runner необходимо производить после того, как веб-интерфейс GitLab станет доступным, так как для регистрации GitLab runner требуется доступ к веб-интерфейсу GitLab.  
Ограничение числа параллельных задач CI/CD в GitLab runner задаётся параметром **gitlab_runner_concurrent** в файле **ansible/group_vars/all.yml**, который устанавливает значение параметра **concurrent** в корневом разделе файла конфигурации GitLab runner **config.toml**.  
Для установки и регистрации GitLab runner выполнить команду:
```shell
ansible-playbook -i inventory.ini 5_install_gitlab_runner.yml
```


## Управление существующей установкой

1. Изменение настроек GitLab производится в файле **ansible/roles/install_gitlab/templates/gitlab.rb.j2**.
Для применения изменений настроек GitLab повторить развёртывание GitLab, выполнив команду
```shell
ansible-playbook -i inventory.ini 4_install_gitlab.yml
```
2. Запуск/перезапуск/остановка GitLab и GitLab runner производятся посредстом systemd:
```shell
systemctl start|stop|restart|status gitlab-docker
systemctl start|stop|restart|status gitlab-runner-docker
```
3. Изменение настроек GitLab runner производится в файле **ansible/roles/install_gitlab_runner/templates/gitlab-runner-docker-compose.yml.j2**.  
Для применения изменений настроек GitLab runner повторить развёртывание GitLab runner, выполнив команду
```shell
ansible-playbook -i inventory.ini 5_install_gitlab_runner.yml
```
4. Изменение имени ветки по-умолчанию (**master**/**main**) для всех проектов производится в разделе **Menu->Admin->Settings->Repository->Default initial branch name** интерфейса администратора GitLab.
5. Срок хранения резервных копий данных и конфигурации GitLab настраивается в разделе **default -> Object Storage -> Бакеты -> backups -> Жизненный цикл** облака Yandex.
6. Подключение к GitLab Docker Registry производится с учётными записями пользователей GitLab по адресу **gitlab.example.com:5050**, например:
```shell
docker login gitlab.example.com:5050
Username: username@example.com
Password: **********
```


## Восстановление GitLab из резервной копии

1. Загрузить из хранилища данных S3 архивы резервной копии конфигурации и данных GitLab на сервер GitLab
2. Распаковать содержимое архива резервной копии _конфигурации_ GitLab в каталог **/opt/gitlab/etc** на сервере GitLab
```shell
tar zxf 1648767601_2022_03_31_gitlab_config_backup.tar.gz -C /
# Проверка:
ls /opt/gitlab/etc
```
3. С локальной машины произвести развёртывание GitLab
```shell
ansible-playbook -i inventory.ini 4_install_gitlab.yml
```
4. Дождаться запуска GitLab -- веб-интерфейс должен отображать страницу входа
5. На серевере GitLab создать каталог резервных копий данных GitLab и переместить в него архив резервной копии _данных_ GitLab
```shell
mkdir -p /opt/gitlab/var/opt/backups
mv 1648767632_2022_03_31_14.9.1_gitlab_backup.tar /opt/gitlab/var/opt/backups
```
6. Подключиться к командной строке docker-контейнера GitLab
```shell
docker exec -it gitlab bash
```
7. В командной строке контейнера GitLab остановить компоненты GitLab, связанные с базой данных
```shell
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
# Проверка:
gitlab-ctl status
```
8. В командной строке docker-контейнера GitLab запустить процесс восстановления данных из резервной копии, при этом имя файла архива резервной копии данных указывается без суффикса **_config_backup.tar.gz**.
```shell
gitlab-backup restore BACKUP=1648767632_2022_03_31_14.9.1
```
На все запросы **Do you want to continue (yes/no)?** ответить **yes**.  
9. Дождаться завершения процеса восстановления данных, затем выполнить переконфигурирование, запуск и проверку целостности данных GitLab
```shell
gitlab-ctl reconfigure
gitlab-ctl restart
gitlab-rake gitlab:check SANITIZE=true
gitlab-rake gitlab:doctor:secrets
gitlab-rake gitlab:artifacts:check
gitlab-rake gitlab:lfs:check
gitlab-rake gitlab:uploads:check
```
10. Выйти из командной строки docker-контейнера GitLab
```shell
exit
```
11. Дождаться завершения запуска веб-интерфейса GitLab и проверить работоспособность GitLab и сохранность данных.
12. Дополнительная информация: https://docs.gitlab.com/ee/raketasks/backup_restore.html#restore-gitlab


## Приложение 1. Параметры GitLab, задаваемые в файле шаблона gitlab.rb.j2
```ruby
external_url 'https://{{ gitlab_server_address }}'
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.yandex.ru"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "{{ gitlab_email_address }}"
gitlab_rails['smtp_password'] = "{{ gitlab_email_password }}"
gitlab_rails['smtp_domain'] = "{{ gitlab_smtp_domain }}"
gitlab_rails['gitlab_email_from'] = "{{ gitlab_email_address }}"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_openssl_verify_mode'] = "peer"
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['Yandex']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_providers'] = [
  {
    "name" => "oauth2_generic",
    "app_id" => "{{ oauth_app_id }}",
    "app_secret" => "{{ oauth_app_secret }}",
    "args" => {
      client_options: {
        "site" => "https://oauth.yandex.ru",
        "authorize_url" => "/authorize",
        "token_url" => "/token",
        "user_info_url" => "https://login.yandex.ru/info",
      },
      user_response_structure: {
        attributes: {
          name: 'login',
          email: 'default_email',
          first_name: 'first_name',
          last_name: 'last_name'
        }
      },
      redirect_url: 'https://{{ gitlab_server_address }}/users/auth/Yandex/callback',
      name: 'Yandex',
      strategy_class: "OmniAuth::Strategies::OAuth2Generic"
    }
  }
]
gitlab_rails['manage_backup_path'] = true
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
gitlab_rails['backup_keep_time'] = 604800
gitlab_rails['backup_upload_connection'] = {
  'provider' => 'AWS',
  'region' => '{{ s3_region }}',
  'aws_access_key_id' => '{{ s3_access_key }}',
  'aws_secret_access_key' => '{{ s3_secret_key }}',
  'endpoint' => '{{ s3_endpoint }}',
  'path_style' => true
}
gitlab_rails['backup_upload_remote_directory'] = 'backups/gitlab'
gitlab_rails['gitlab_shell_ssh_port'] = 2222
gitlab_rails['initial_shared_runners_registration_token'] = "{{ gitlab_runner_initial_registration_token }}"
gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "{{ gitlab_server_address }}"
gitlab_rails['registry_port'] = "5050"
gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['{{ gitlab_email_address }}']
letsencrypt['group'] = 'root'
letsencrypt['key_size'] = 2048
letsencrypt['owner'] = 'root'
letsencrypt['wwwroot'] = '/var/opt/gitlab/nginx/www'
letsencrypt['auto_renew'] = true
letsencrypt['auto_renew_hour'] = 3
letsencrypt['auto_renew_minute'] = 10
letsencrypt['auto_renew_day_of_month'] = "*/7"
letsencrypt['auto_renew_log_directory'] = '/var/log/gitlab/lets-encrypt'
```


## Приложение 2. Настройка Nginx reverse proxy для работы Let's Encrypt  
При выдаче сертификата SSL служба Let's Encrypt обращается к web-серверу по URI `/.well-known/acme-challenge/` в целях проверки принадлежности домена web-серверу.  

Если сервер GitLab доступен только из локальной сети и недоступен из сети Интернет, но, при этом, имеет доменное имя в сети Интернет и должен иметь сертификат SSL, то необходимо обеспечить доступ серверов службы Let's Encrypt к порту 80/tcp сервера GitLab.

Доступ Let's Encrypt к серверу GitLab может быть реализован посредством NAT port forwarding на граничном маршрутизаторе локальной сети или посредстом reverse proxy на web-сервере, подключенном к сети Интернет.

Ниже приведён пример настройки reverse proxy на веб-сервере Nginx.  
```conf
server {
    listen 80;

    server_name gitlab.example.com;
    access_log /var/log/nginx/gitlab.example.com.access_log;
    error_log /var/log/nginx/gitlab.example.com.error_log info;

    # Let's Encrypt
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";

        proxy_pass http://gitlab.example.com; # GitLab server internal address
        proxy_read_timeout 3600s;
        proxy_http_version 1.1;

        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        allow all;
    }

    location / {
        deny all;
    }
}
```
