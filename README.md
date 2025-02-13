# Jenkins shared library for 1C:Enterprise 8

## Цель

Создание библиотеки (или плагина) для Jenkins, позволяющей:

* максимально упростить написание Jenkinsfile для процесса CI в условиях платформы 1С:Предприятие 8
* иметь схожий и контролируемый пайплайн для всех проектов
* дать пользователю в руки простой декларативный конфигурационный файл, вместо требования описывать всю сложную логику по работе с 1С 

## Общие положения

* в активной разработке и поиске "своего пути" по разработке библиотеки;
* формат конфигурационного файла **не** стабилизирован;
* обратная совместимость **пока** не гарантируется, внимательно читайте changelog;
* количество stage будет со временем увеличиваться;
* использовать на свой страх и риск;
* любая помощь приветствуется.

## Ограничения

1. Для шага подготовки требуется любой агент с меткой `agent`.
1. Для запуска шага анализа SonarQube требуется агент с меткой `sonar`.
1. Для запуска шага валидации EDT требуется агент с меткой `edt` (для собственно валидации) и агент с меткой `oscript` (для трансформации результатов с помощью библиотеки [stebi](https://github.com/Stepa86/stebi)).
1. Для запуска шагов, работающих с 1С (подготовка, синтаксический контроль и т.д.) требуется агент с меткой, совпадающей со значением в поле `v8version` файла конфигурации.
1. В качестве ИБ используется файловая база, создаваемая в `./build/ib` на основании конфигурации из хранилища без пользователей. При необходимости вы можете создать пользователей на фазе инициализации ИБ.
1. Stage "Дымовые тесты" пока пустой.
1. Запуск `vrunner` на текущий момент происходит из локального каталога `oscript_modules`. Предполагается наличие в корне репозитория файла `packagedef`, в котором бы была указана зависимость от `vanessa-runner`

## Возможности

1. Все шаги можно запустить на базе docker-образов из форка репозитория onec-docker. См. [памятку по слоям и последовательности сборки](https://github.com/firstBitSemenovskaya/onec-docker/blob/feature/first-bit/Layers.md)
1. Подготовка информационной базы по версии из хранилища конфигурации.
1. Запуск ИБ в режиме выполнения обработчиков обновления БСП.
1. Дополнительные шаги инициализации данных в ИБ.
1. Трансформация кода из формата конфигуратора в формат EDT (только если включен шаг `edtValidate`).
1. Запуск BDD сценариев с сохранением результатов в формате Allure.
1. Запуск синтаксического контроля средствами конфигуратора и сохранение результатов в виде отчета jUnit.
1. Запуск валидации проекта средствами EDT и конвертация отчета в формате generic issues.
1. Запуск статического анализа для SonarQube.
1. Публикация результатов junit и Allure в интерфейс Jenkins.
1. Конфигурирование логгера запускаемых oscript-приложений.

## Подключение

Инструкция по подключению библиотеки: https://jenkins.io/doc/book/pipeline/shared-libraries/#using-libraries

## Примеры Jenkinsfile

Если в настройках подключения shared-library включен флаг "Load implicitly":

```groovy
pipeline1C()
```

В обратном случае:

```groovy
@Library('jenkins-lib') _

pipeline1C()
```

> Да, вот и весь пайплайн. Конфигурирование через json.

## Внешний вид пайплайна в интерфейсе Blue Ocean

![image](https://user-images.githubusercontent.com/1132840/121659170-b67f9e00-caaa-11eb-91be-107a689e893d.png)

## Конфигурирование

По умолчанию применяется [файл конфигурации из ресурсов библиотеки](resources/globalConfiguration.json)

Поверх него накладывается конфигурация из файла `jobConfiguration.json` в корне проекта, если он присутствует.

Пример переопределения:

* указывается точная версия платформы (и соответственно метка агента, см. ограничения)
* идентификаторы credentials для пути к хранилищу и к паре логин/пароль для авторизации в хранилище (необходимы, если применяются шаги, работающие с информационной базой)
* включаются шаги запуска статического анализа SonarQube, валидации средствами EDT и синтаксического контроля 

```json
{
    "$schema": "https://raw.githubusercontent.com/firstBitSemenovskaya/jenkins-lib/master/resources/schema.json",
    "v8version": "8.3.14.1976",
    "secrets": {
        "storagePath": "f7b21c02-711a-4883-81c5-d429454e3f8b",
        "storage" : "c1fc5f33-67d4-493f-a2a4-97d3040e4b8c"
    },
    "stages": {
        "sonarqube": true,
        "edtValidation": true,
        "syntaxCheck": true
    }
}
```

## Параметры по умолчанию

В библиотеке применяется принцип "соглашения по конфигурации" (convention over configuration): конфигурационный файл
содержит ряд настроек "по умолчанию". При соблюдении определенных правил структуры репозитория можно сократить
количество переопределений конфигурационного файла до минимума.

* Общее:
  * В качестве маски версии платформы используется строка "8.3".
  * Исходники конфигурации ожидаются в каталоге `src/cf`.
  * TODO: Имена "секретов" (jenkins credentials) по умолчанию высчитываются как `GROUP_REPO_KEY`, где `GROUP` и `REPO` - это группа проектов и имя проектов (например, `firstBitSemenovskaya` и `jenkins-lib`), а `KEY` - ключ секрета:
    * `STORAGE_PATH` - путь к хранилищу конфигурации;
    * `STORAGE_USER` - параметры авторизации в хранилище вида "username with password".
  * Все "шаги" по умолчанию выключены.
  * Результаты в формате `allure` ожидаются в каталоге `build/out/allure` или его подкаталогах.
* Инициализация:
  * Если включен шаг `initSteps`, то будет выполняться запуск ИБ с целью запуска обработчиков обновления из БСП. (`initInfobase` -> `runMigration`)
  * Если в настройках шага инициализации не заполнен массив дополнительных шагов миграции (`initInfobase` -> `additionalInitializationSteps`), но в каталоге `tools` присутствуют файлы с именами, удовлетворяющими шаблону `vrunner.init*.json`, то автоматически выполняется запуск `vrunner vanessa` с передачей найденных файлов в качестве значения настроек (параметр `--settings`) в порядке лексиграфической сортировки имен файлов.
* BDD:
  * Если в конфигурационном файле проекта не заполнена настройка `bdd` -> `vrunnerSteps`, то автоматически выполняется запуск `vrunner vanessa --settings tools/vrunner.json`.
* Синтаксический контроль:
  * Если в репозитории существует файл `tools/vrunner.json`, то синтаксический контроль конфигурации с помощью конфигуратора будет выполняться с передачей файла в параметры запуска `vrunner syntax-check --settings tools/vrunner.json`.
  * TODO: Значение параметра `--mode` из конфигурационного файла vrunner имеют приоритет над `syntaxCheck` -> `checkModes`. Значение из `jobConfiguration.json` передается только в том случае, если параметр отсутствует в конфигурационном файле `vrunner`.
* Трансформация результатов валидации EDT:
  * По умолчанию из результатов анализа исключаются замечания, сработавшие на модулях с включенным запретом редактирования (желтый куб с замком)
