# TW-101163: Maven User Settings в TeamCity Pipelines

Минимальный Maven-проект для проверки https://youtrack.jetbrains.com/issue/TW-101163.
Требования: Maven 3.6.3+, JDK 11+, доступ агента к Maven Central для плагинов.

## Как работает проверка

В фазе `validate` Maven Enforcer проверяет свойство `tw101163.settings.marker`.
Оно определяется только активным профилем выбранного settings.xml.
По умолчанию ожидается `project-settings`. Отсутствующий или неверный маркер
приводит к BUILD FAILURE с сообщением TW-101163. `verify` дополнительно компилирует Java-пример.
Значение маркера не зависит от кеша зависимостей, поэтому прогретый Maven-кеш
не скрывает отсутствие settings.xml. Не передавайте проверяемое свойство через `-D`
или параметры TeamCity — это обойдёт проверку.

## Подготовка TeamCity

1. Используйте сборку с реализацией TW-101163. Проверьте, что включён флаг
   `teamcity.pipelines.feature.mavenSettings.enabled` (по комментарию в задаче он не включён по умолчанию).
2. Создайте тестовый родительский проект и дочерний проект. В родительском загрузите
   `settings/settings-parent.xml` через Project Settings → Maven Settings.
3. В дочернем загрузите `settings/settings-project.xml`. Опционально загрузите
   `settings/settings-root.xml` в Root project для проверки наследования от корня.
   Если Root менять неудобно, достаточно сценариев с родителем и дочерним проектом.
4. Подключите Git-репозиторий с содержимым этой папки в корне.
5. В дочернем проекте создайте Pipeline с Maven step, POM `pom.xml`, Goals `verify`,
   JDK 11+ и Maven 3.6.3+. В User Settings выберите загруженный settings-project.xml.
6. Создайте классическую Build Configuration в том же проекте с тем же VCS root,
   Maven step, POM и Goals. Выберите тот же файл в User settings selection.

## Сценарии

Для каждого сценария выберите User Settings через UI. Опции `-s` ниже нужны
только для локального запуска: не добавляйте их вручную в Pipeline, иначе
вы не проверите, передаёт ли настройки сам TeamCity.

| Сценарий | User Settings | Goals / дополнительные аргументы | Ожидание |
|---|---|---|---|
| Файл текущего проекта | Загруженный settings-project.xml | `verify` | SUCCESS |
| Наследование от родителя | Загруженный settings-parent.xml | `verify -Dexpected.settings.marker=parent-settings` | SUCCESS |
| Настройки Root (опционально) | Загруженный settings-root.xml | `verify -Dexpected.settings.marker=root-settings` | SUCCESS |
| Файл из репозитория | Custom path: `settings/settings-custom.xml` | `verify -Dexpected.settings.marker=custom-settings` | SUCCESS |
| Без настроек | Default / без выбранного файла | `verify` | FAILURE: marker is missing, если маркера нет в агентских settings |
| Неверный файл | Загруженный settings-parent.xml | `verify` | FAILURE: wrong settings file |
| Пустой файл | Custom path: `settings/settings-empty.xml` | `verify` | FAILURE: marker is missing |
| Переключение выбора | Сначала project, затем parent | Менять ожидаемый маркер соответственно | SUCCESS в обоих запусках |
| Классическая конфигурация | Загруженный settings-project.xml | `verify` | SUCCESS |

При запуске без настроек убедитесь, что агентский ~/.m2/settings.xml не содержит
этот тестовый маркер. Пустой файл даёт более контролируемую негативную проверку.

Проверьте, что выпадающий список показывает файлы текущего проекта и предков,
группирует их по проектам, а Manage settings files открывает Maven Settings
нужного проекта. После сохранения и повторного открытия выбранный файл должен сохраниться.
Для проверки изоляции можно создать соседний проект и убедиться, что его файлы
не предлагаются в текущем проекте.

В build log сравните Maven command line у Pipeline и классической конфигурации:
для выбранного серверного файла ожидается `-s <временный settings.xml>`;
в исходном описании также указан `-Dteamcity.maven.userSettings.path=<...>`.
Конкретное имя временного файла изменяется. Сохраняйте лог и результат Enforcer:
одна только видимость файла в списке не доказывает, что Maven его применил.

Проект проверяет передачу и чтение settings.xml через активный профиль.
Он не проверяет отдельно аутентификацию к приватному репозиторию или работу mirrors.

## Локальная проверка

Запускайте команды из корня репозитория:

```sh
# Успех
mvn -B -ntp -s settings/settings-project.xml verify
mvn -B -ntp -s settings/settings-parent.xml -Dexpected.settings.marker=parent-settings validate
mvn -B -ntp -s settings/settings-custom.xml -Dexpected.settings.marker=custom-settings validate
mvn -B -ntp -s settings/settings-root.xml -Dexpected.settings.marker=root-settings validate

# Ожидаемые ошибки
mvn -B -ntp -s settings/settings-empty.xml validate
mvn -B -ntp -s settings/settings-parent.xml validate
```

YAML намеренно не задан: создайте Maven step через UI целевой версии TeamCity,
выберите User Settings и проверьте сгенерированный YAML. Это позволит проверить
реальное имя и сериализацию поддерживаемого поля без предположений о схеме.
