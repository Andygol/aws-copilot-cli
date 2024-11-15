---
title: "Сервіси"
---

# Сервіси

Одна з чудових особливостей контейнерів полягає в тому, що коли ви написали свій код, запустити його локально так само просто, як набрати `docker run`. Copilot робить запуск тих самих контейнерів на AWS таким же простим, як введення `copilot init`. Copilot збере ваш образ, завантажить його в Amazon ECR і налаштує всю інфраструктуру для запуску вашого сервісу у масштабованому та безпечному режимі.

## Створення Сервісу

Створення сервісу для запуску ваших контейнерів на AWS можна здійснити кількома способами. Найпростіший спосіб — виконати команду `init` з теки з вашим Dockerfile.

```console
copilot init
```

Вас запитають, до якого застосунку ви хочете додати цей сервіс (або запропонують створити застосунок, якщо його ще немає). Copilot потім запитає про __тип__ сервісу, який ви намагаєтеся створити.

Після вибору типу сервісу, Copilot виявить будь-які перевірки справності або відкриті порти з вашого Dockerfile і запитає, чи хочете ви розгорнути.

## Вибір типу Сервісу

Ми вже згадували, що Copilot налаштує всю інфраструктуру, необхідну для запуску вашого сервісу. Але як він знає, який тип інфраструктури використовувати?

Коли ви налаштовуєте сервіс, Copilot запитає вас про те, який тип сервісу ви хочете створити.

### Інтернет-орієнтовані сервіси

Якщо ви хочете, щоб ваш сервіс обслуговував публічний інтернет-трафік, у вас є три варіанти:

* "Запит-орієнтований вебсервіс" створить сервіс AWS App Runner для запуску вашого сервісу.
* "Статичний сайт" створить спеціальний дистрибутив CloudFront та S3 кошик для вашого статичного вебсайту.
* "Вебсервіс з балансуванням навантаження" створить Application Load Balancer, Network Load Balancer або обидва, разом з групами безпеки, сервісом ECS на Fargate для запуску вашого сервісу.

#### Запит-орієнтований вебсервіс <a id="request-driven-web-service"></a>

Сервіс AWS App Runner, який автоматично масштабує ваші екземпляри на основі вхідного трафіку і зменшує до базової кількості екземплярів, коли немає трафіку. Цей варіант є більш економічно вигідним для HTTP сервісів з раптовими сплесками обсягів запитів або низькими обсягами запитів.

На відміну від ECS, сервіси App Runner стандартно не підключені до VPC. Щоб маршрутизувати вихідний трафік через VPC, ви можете налаштувати поле [`network`](../../manifest/rd-web-service/#network) у маніфесті.

#### Статичний сайт <a id="static-site"></a>

Статичний вебсайт, розміщений у S3, який обслуговується розподілом Amazon CloudFront. Copilot завантажує ваші статичні активи в новий S3 кошик, налаштований для розміщення статичного вебсайту. Кешування з [CloudFront Content Delivery Network (CDN)](../../developing/content-delivery/) оптимізує вартість і швидкість. З кожним повторним розгортанням попередній кеш анулюється.

#### Вебсервіс з балансуванням навантаження <a id="load-balanced-web-service"></a>

Сервіс ECS, що запускає завдання на Fargate з Application Load Balancer, Network Load Balancer або обома як вхідними точками. Цей варіант підходить для HTTP або TCP сервісів зі стабільними обсягами запитів, які потребують доступу до ресурсів у VPC або вимагають розширеної конфігурації.

Зверніть увагу, що керований Copilot Application Load Balancer є ресурсом на рівні середовища і використовується всіма вебсервісами з балансуванням навантаження в межах середовища. Починаючи з версії v1.32.0, у вас є можливість імпортувати наявний ALB на рівні сервісу, вказавши його у вашому [маніфесті робочого навантаження](../../manifest/lb-web-service/#http-alb). Щоб дізнатися більше, перейдіть [сюди](../environments/#load-balancers-and-dns). На відміну від цього, Network Load Balancer є ресурсом на рівні сервісу і тому не використовується спільно між сервісами.

Нижче наведена діаграма для вебсервісу з балансуванням навантаження, який включає лише Application Load Balancer.

![lb-web-service-infra](https://user-images.githubusercontent.com/879348/86045951-39762880-ba01-11ea-9a47-fc9278600154.png)

### Сервіс бекенду <a id="backend-service"></a>

Якщо ви хочете створити сервіс, який не може бути доступний зовні, а лише з інших сервісів у вашому застосунку, ви можете створити __Сервіс бекенду__. Copilot створить сервіс ECS, що працює на AWS Fargate, але не налаштує жодних інтернет-орієнтованих точок доступу. Балансування навантаження доступне для сервісів бекенду. Щоб дізнатися про створення внутрішніх балансувальників навантаження, перейдіть [сюди](../../developing/internal-albs/).

![backend-service-infra](https://user-images.githubusercontent.com/879348/86046929-e8673400-ba02-11ea-8676-addd6042e517.png)

### Сервіс робочих навантажень <a id="worker-service"></a>

__Сервіси робочого навантаження__ дозволяють реалізувати асинхронну комунікацію між сервісами за допомогою [архітектур pub/sub](https://aws.amazon.com/pub-sub-messaging/). Мікросервіси у вашому застосунку можуть `публікувати` події до [топіків Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html), які потім можуть бути спожиті "Сервісом робочого навантаження".

Сервіс робочого навантаження складається з:

* Однієї або більше [черг Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) для обробки сповіщень, опублікованих у топіках, а також [черг для мертвих листів](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) для обробки збоїв.
* Сервісу Amazon ECS на AWS Fargate, який має дозвіл на опитування черг SQS і асинхронну обробку повідомлень.

![worker-service-infra](https://user-images.githubusercontent.com/25392995/131420719-c48efae4-bb9d-410d-ac79-6fbcc64ead3d.png)

## Конфігурація та Маніфест

Після того, як ви виконали `copilot init`, ви могли помітити, що Copilot створив файл з назвою `manifest.yml` у теці `copilot/[назва сервісу]/`. Цей файл маніфесту містить загальні параметри конфігурації для вашого сервісу. Хоча точний набір параметрів залежить від типу сервісу, який ви запускаєте, загальні параметри включають ресурси, виділені для вашого сервісу (наприклад, памʼять і процесор), перевірки справності та змінні середовища.

Розгляньмо маніфест для вебсервісу з балансуванням навантаження з назвою _front-end_.

```yaml
name: front-end
type: Load Balanced Web Service

image:
  # Шлях до Dockerfile вашого сервісу.
  build: ./Dockerfile
  # Порт, відкритий у ваш контейнер для маршрутизації трафіку до нього.
  port: 8080

http:
  # Запити до цього шляху будуть перенаправлені до вашого сервісу.
  # Щоб відповідати всім запитам, ви можете використовувати шлях "/".
  path: '/'
  # Ви можете вказати користувацький шлях перевірки справності. Стандартно "/"
  # healthcheck: '/'

# Кількість одиниць CPU для завдання.
cpu: 256
# Кількість памʼяті в MiB, що використовується завданням.
memory: 512
# Кількість завдань, які повинні працювати у вашому сервісі.
count: 1

# Необовʼязкові поля для більш складних випадків використання.
#
variables:                    # Передача змінних середовища у вигляді пар ключ-значення.
  LOG_LEVEL: info

#secrets:                         # Передача секретів з AWS Systems Manager (SSM) Parameter Store.
#  GITHUB_TOKEN: GH_SECRET_TOKEN  # Ключ - це назва змінної середовища,
                                  # значення - це назва параметра SSM.

# Ви можете перевизначити будь-яке з значень, визначених вище, для кожного середовища.
environments:
  prod:
    count: 2               # Кількість завдань для запуску в середовищі "prod".
```

Щоб дізнатися про специфікацію файлів маніфесту, дивіться сторінку [manifest](../../manifest/overview/).

## Розгортання Сервісу

Після того, як ви налаштували свій сервіс, ви можете розгорнути його (і будь-які зміни до вашого маніфесту) за допомогою команди розгортання:

```console
copilot deploy
```

Виконання цієї команди:

1. Збере ваш образ локально
2. Завантажить його в репозиторій ECR вашого сервісу
3. Перетворить ваш файл маніфесту в CloudFormation
4. Упакує будь-яку додаткову інфраструктуру в CloudFormation
5. Розгорне ваш оновлений сервіс і ресурси в CloudFormation

Якщо у вас є кілька середовищ, вам буде запропоновано вибрати, до якого середовища ви хочете виконати розгортання.

## Дослідження вашого Сервісу

Тепер, коли ми запустили сервіс, ми можемо перевірити його за допомогою Copilot. Нижче наведено кілька поширених способів перевірки вашого розгорнутого сервісу.

### Що у вашому сервісі?

Виконання `copilot svc show` покаже вам опис вашого сервісу. Ось приклад виводу, який ви можете побачити для __Вебсервісу з балансуванням навантаження__. Цей вивід включає конфігурацію вашого сервісу для кожного середовища, будь-які налаштовані сигнали відкату, всі точки доступу вашого сервісу та змінні середовища і секрети, передані у ваш сервіс. Ви також можете надати необовʼязковий прапорець `--resources`, щоб побачити всі ресурси AWS, повʼязані з вашим сервісом.

```console
$ copilot svc show
About

  Application       my-app
  Name              front-end
  Type              Load Balanced Web Service

Configurations

  Environment       Tasks               CPU (vCPU)          Memory (MiB)        Port
  -----------       -----               ----------          ------------        ----
  test              1                   0.25                512                 80

Rollback Alarms

  Name                              Environment  Description
  ----                              -----------  -----------
  my-app-test-front-end-CopilotRol  test         Roll back ECS service if CPU utilizat
  lbackCPUAlarm                                  ion is greater than or equal to 50% t
                                                 wice in 3 minutes.

Routes

  Environment       URL
  -----------       ---
  test              http://my-ap-Publi-1RV8QEBNTEQCW-1762184596.ca-central-1.elb.amazonaws.com

Internal Service Endpoints

  Endpoint                          Environment  Type
  --------                          -----------  ----
  front-end:80                      test         Service Connect
  front-end.test.my-app.local:8080  test         Service Discovery

Variables

  Name                                Container  Environment  Value
  ----                                ---------  -----------  -----
  COPILOT_APPLICATION_NAME            front-end  test         my-app
  COPILOT_ENVIRONMENT_NAME              "        test         test
  COPILOT_LB_DNS                        "        test         my-ap-Publi-1RV8QEBNTEQCW-1762184596.ca-central-1.elb.amazonaws.com
  COPILOT_SERVICE_DISCOVERY_ENDPOINT    "        test         test.my-app.local
  COPILOT_SERVICE_NAME                  "        test         front-end

Secrets

  Name                   Container  Environment  Value
  ----                   ---------  -----------  -----
  GITHUB_WEBHOOK_SECRET  front-end  test         parameter/GH_WEBHOOK_SECRET
```

Вивід `copilot svc show` змінюється залежно від типу вашого сервісу. Наприклад, опис для __Статичного сайту__ включає дерево вмісту вашого S3 кошику.

```console
% copilot svc show
Service name: static-site
About

  Application  my-app
  Name         static-site
  Type         Static Site

Routes

  Environment  URL
  -----------  ---
  test         https://d399t9j1xbplme.cloudfront.net/

S3 Bucket Objects

  Environment  test
.
├── ReadMe.md
├── error.html
├── index.html
├── Images
│   ├── SomeImage.PNG
│   └── AnotherImage.PNG
├── css
│   ├── Style.css
│   ├── all.min.css
│   └── bootstrap.min.css
└── images
    └── bg-masthead.jpg
```

### Який статус вашого сервісу?

Часто корисно мати можливість перевірити статус вашого сервісу. Чи всі екземпляри мого сервісу справні? Чи є якісь сигнали тривоги? Для цього ви можете виконати `copilot svc status`, щоб отримати зведений статус вашого сервісу.

```console
$ copilot svc status
Service: front-end
Task Summary

  Running   ██████████  1/1 desired tasks are running
  Health    ██████████  1/1 passes HTTP health checks
            ██████████  1/1 passes container health checks

Tasks

  ID        Status      Revision    Started At     Cont. Health  HTTP Health
  --        ------      --------    ----------     ------------  -----------
  37236ed3  RUNNING     9           12 minutes ago HEALTHY       HEALTHY

Alarms

  Name                            Type          Condition                       Last Updated    Health
  ----                            ----          ---------                       ------------    ------
  TargetTracking-service/my-app-  Auto Scaling  CPUUtilization > 70.00 for 3 d  5 minutes ago   OK
  test-Cluster-0jTKWTNBKviF/my-a                atapoints within 3 minutes
  pp-test-front-end-Service-r5h6
  hMZVbWkz-AlarmHigh-f0f31c7b-74
  61-415c-9dfd-81b983cbe0df

  TargetTracking-service/my-app-  Auto Scaling  CPUUtilization < 63.00 for 15   5 minutes ago   ALARM
  test-Cluster-0jTKWTNBKviF/my-a                datapoints within 15 minutes
  pp-test-front-end-Service-r5h6
  hMZVbWkz-AlarmLow-698f9f17-6c0
  c-4db1-8f1d-e23de97f5459
```

 Так само як і з `copilot svc show`, вивід `copilot svc status` змінюється залежно від типу сервісу. Наприклад, вивід для __Запит-орієнтованого вебсервісу__ включає системні журнали, а вивід для __Статичного сайту__ включає кількість обʼєктів у кошику S3 та їх розмір.

### Де знаходяться журнали мого сервісу?

Перевірка журналів вашого сервісу також проста. Виконання `copilot svc logs` покаже вам найновіші журнали вашого сервісу. Ви можете слідкувати за журналами в режимі реального часу за допомогою прапорця `--follow`.

```console
$ copilot svc logs
37236ed 10.0.0.30 🚑 Health-check ok!
37236ed 10.0.0.30 🚑 Health-check ok!
37236ed 10.0.0.30 🚑 Health-check ok!
37236ed 10.0.0.30 🚑 Health-check ok!
37236ed 10.0.0.30 🚑 Health-check ok!
```

!!! info "Інформація"
    Журнали недоступні для сервісів Статичного сайту.
