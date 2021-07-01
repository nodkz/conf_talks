# Тезисы (DRAFT):

## 1. Нужен если:

- взаимодействие между бэком и фронтом
  - GraphQL разрабатывался для удобства фронтендеров
  - в SPA и мобилках желательно использовать "умные стейт-менеджеры", типа Apollo Client, Relay и пр., которые дико упрощают выполнение запросов, кешируют и нормализовывают данные
- мульён микросервисов со своими ендпоинтами
  - когда список ендпоинтов начинает идти на сотни, то знание о них и связями между ними можно заложить GraphQL. Зачем мучить бедных фронтендеров безумным пластом знаний, учить их тому как устроена наша бэкендерская кухня, которая и без того постоянно меняется.
- нужны тонкие клиенты на фронтенде согласно серверной схеме данных
  - graphql-codegen может прекрасно генерировать фетчеры данных на базе GraphQL-запросов (не путать с GraphQL-схемой. Т.е. если для OpenAPI, json-rpc генерируются полные клиенты со всеми методами, типами и полями. То с GraphQL можно генерировать только то, что есть в GraphQL-запросах, пропуская тысячи реально неиспользуемых типов и полей).
- нужна статическая типизация
  - к примеру graphql-codegen генерирует typescript дефинишены согласно GraphQL-запросов, и если поменяется тип у поля на бэке, либо оно удалиться, то любое неправильное потребление полей можно будет отловить на билде приложения
- если вам тесно в парадигме 4 операций CRUD в RESTfull API (изменения состояния)
  - например операции "провести оплату", "приостановить подписку" и пр. достаточно плохо ложатся в CRUD
  - тут нужно смотреть в сторону RPC (отправляя на пенсию HTTP-методы). При этом если вы смотрите в сторону json-rpc, то лучше рассмотрите GraphQL. Они практически идентичны, только у GraphQL, есть возможно вызвать получение вложенных данных на результате выполнения родительской операции; удобная браузерная IDE для написания запросов и их проверке; побогаче туллинг кодогенерации.
- если много разных entity (models), между ними много связей и клиенту нужны их агрегации (а-ля LEFT JOIN) и желательно за один http-запрос.
  - часто удобно для всяких админок и дашбордов на клиентских приложениях
  - но даже некоторые бэкенды (сервисы) общаются через GraphQL, так как очень много связей между Entity и GraphQL неплохо передает эти знания через свою схему, и написать самостоятельно сложную выборку данных из чужого сервиса достаточно просто
  - когда много разработчиков, нужен какой-то инструмент для хранения знаний о DataDomain и язык общения. У GraphQL ytgkj
- у вас большие Entity как у GitHub, и таскать все 100 полей, когда нужно всего два поля слишком затратно
  - также graphql-codegen позволяет генерировать удобные fetche'ы (react hooks), которые содержать в себе только используемы поля. Если схема на 10000 типов, то в рантайм и сборку попадут только используемые типы и поля (в отличие от OpenApi, json-rpc где генераторы клиентов создают клиента с полным описанием всех методов, типов и полей.)

## 2. Не нужен, если:

- Работа с файлами или другими бинарными данными
  - GraphQL создавался для типизированных структур, бинарные данные он может передавать только в виде base64 строки. А это увеличивает более чем на 30% исходный размер файла. Тратим сеть, память, процессор. И можем передать файл не более 380MB (т.к. в NodeJS ограничение на строку в 500MB)
  - Альтернатива, это микс GraphQL и http-multipart <https://github.com/jaydenseric/graphql-multipart-request-spec>. Правда это уже "франкенштейн" из GraphQL и Multipart из REST API. Тут много что делает Server, который тюнит транспорт.
  - Используйте S3 signed-urls для загрузки бинарных данных через REST. GraphQL может передавать эти ссылки, но не сами бинарные данные. <https://github.com/nodkz/conf-talks/tree/master/articles/graphql/fileUploads>
  - Но никогда не говори никогда: мы вынуждены передавать base64 файлы при отправки писем со счетами на оплату. Сервис биллинга формирует уведомление, а вот сервис уведомлений уже делает отправку писем со вложениями.
- CRDT и многопользовательское редактирование с версионированием
  - Conflict-free Replicated Data Types
  - Это целая наука разбирать конфликты и синхронизировать состояние. Вы конечно можете GraphQL прикрутить со стороны, но боюсь что вы провалитесь в прекрасный мир типизации всего этого добра. Проще взять готовые решения, чем крутить свой велосипед.
  - Ну нет толком никаких алгоритмов в GraphQL <https://arxiv.org/pdf/1608.03960.pdf> take fig.9 as an example
- Аутентификация
  - Ох сколько копий мы поломали, пытаясь запихнуть ввод логина и пароля через GraphQL.
  - Операция `login` – это Query или Mutation?
  - Хорошо, а это законно в резолвере устанавливать куку пользователю? Ведь GraphQL вроде как протоколо-независим. (У нас есть GraphQL-метод loginAsClient, он устанавливает куку)
  - А еще сброс пароля по ссылке, двухфакторка, oidc (oauth2)
  - В любом случае большинство современных систем аутентификации построены поверх http c кучей редиректов. Работа с http, редиректами, куками – это не про GraphQL.
  - И зачем вам нужна типизация при вводе логина и пароля от GraphQL – не понятно. Проще сервис аутентификации пелить без GraphQL.
  - А вот к примеру просмотр и редактирование профиля клиента, уже вполне хорошо ложиться в GraphQL. Тут просто тупая работа с типизированными данными.
  - Немного про Индетификацию/Авторизацию
    - Проще сделать весь графкуэль защищенным на уровне формирования контекста, чем в каждом резолвере индетифицировать пользователя. Ведь где-то точно пропустите проверку, и данные получат неиндетифицированные пользователи. Ну авторизацию (проверку прав), вам все равно придется крутить в каждом резолвере. Но это дело могут облегчить миддлвары для резолверов.
    - Заводить два GraphQL, отдельно для неавторизированных пользователей, отдельно для авторизированных.
- Активное межсерверное взаимодействие требовательное к скорости, памяти и трафику
  - Protobuf отличное решение для межсерверного общения. Имея много разных типов их можно хорошо сжимать. А у JSON даже Int нормального нет и тупо гзипится(бротлится) строка с кучей повторов строковых ключей.
  - У вас в проде крутится сотни серверов? Тогда бизнесу становится выгоднее платить разработчику, чтоб он занимался сокращением байтов и тактов. И в gRPC/protobuf это неплохо уже отточено.
- Стримминг данных
  - Хоть в GraphQL есть Subscriptions, которыми удобно пользоваться через Apollo Client или Relay. То вот стримить например котировки для бирж, выглядит сомнительным решением – уж больно много накладных расходов как по сети, так и по CPU.
- Сложные формы с вариативными инпутами (union типы)
  - Для какого-то аргумента вы хотите передать строку или сложный объект. Передать такой аргумент можно используя тип JSON, но у вас не будет статической типизации. 
  - Например в Контуре есть команда у которой 2000 сложных форм, и там используют DTO с кодогенерацией, и такой подход в разы удобнее и безопаснее. Правда можно накостылить стороннюю валидацию JSON типов в GraphQL, но с ней не будет взаимодействовать большинство существующего инструментария - кодогенерация, линтинг, автоподстановка полей в GraphiQL.
  - В GraphQL нет InputUnionType. [Спецификация](https://github.com/graphql/graphql-spec/blob/main/rfcs/InputUnion.md) по ним уже обсуждается года три. Когда она выйдет в свет, то скорее всего этот пункт станет неактуальным.

## 3. А что делать если мы попадаем в обе категории? Где-то подходит GraphQL, а где-то нет.

- Пилите два, три апи на разных технологиях. "Надо дать так, чтоб клиентам было удобно взять". Любое апи это фасад (контроллеры) перед сервисами и моделями. И напилякать их не так уж трудозатратно, если бизнес-логика у вас вынесена на свой слой абстракции и разные фасады (имплементации апи) переиспользуют её.
  - Например админы всяких мониторингов и сборщики метрик, любят старый добрый REST API (меньше писать всяких заголовков)
  - Аналитики и data-сайнтистам подавай возможность писать SQL с LEFT JOIN'ами (здесь Graphql может выступить прекрасной альтернативой)

## 4. Code first или Schema First?

- Schema First – это когда вы пишите схему (обычно в SDL формате), а потом натягиваете на нее резолверы
  - Быстрый старт
  - Для маленьких и средних приложений
  - На длинной дистанции, могут быть проблемы с рефакторингом и внесением изменений
- Code first – это когда у вас есть модели и сервисы, и уже из них генерируется либо пишутся с нуля GraphQL-типы и схема
  - Больше трудозатрат на написание своего генератора, либо поиск готового решения
    - Пример: не надо руками писать все Input-типы, если они по структуре одинаковы с Output-типами (не тратим время на синхронизацию)
    - Пример: graphql-compose-mongoose на одной модели генерирует более 60 GraphQL-типов.
  - На длинной дистанции, вы можете схему менять логику генератора и ваша схема будет меняться вне зависимости от размера схемы и кол-ва типов
- Если у вас гексагональная (или луковая) архитектура, есть DTO, куча интерфейсов, портов и адаптеров, увлекаетесь рефлексией и декораторами, то однозначно надо брать Code first. Зачем писать GraphQL-схему ручками через Schema First, когда вам достаточно артефактов, чтобы ее сгенерировать.

## 5. Итого

Помним что GraphQL это инструмент, а не серебряная пуля. И каждый инструмент удобен под свою категорию задач. Молотком шуруп не закрутишь, отверткой гвоздь не забьешь - это практически нереально, хотя я полагаю, умельцы найдутся.