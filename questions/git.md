## Git

1. Что такое GitFlow?

<details>
  <summary>Ответ</summary>

GitFlow - модель ветвления Git.

*Ключевые идеи*:
1. Данная модель отлично подходит для организации рабочего процесса на основе релизов,
2. Gitflow предлагает создание отдельной ветки для исправлений ошибок в продуктовой среде.

*Последовательность работы при использовании модели Gitflow*:

1. Из *master* создается ветка *develop*.
2. Из *develop* создаются ветки *feature*.
3. Когда разработка новой функциональности завершена, она объединяется с веткой *develop*.
4. Из *develop* создается ветка *release*.
5. Когда ветка релиза готова, она объединяется с *develop* и *master*.
6. Если в *master* обнаружена проблема, из нее создается ветка *hotfix*.
7. Как только исправление на ветке *hotfix* завершено, она объединяется с *develop* и *master*.

</details>

2. Чем `merge` отличается от `rebase`?

<details>
  <summary>Ответ</summary>

- `git merge` - выполняет слияние коммитов из одной ветки в другую. В этом процессе изменяется только целевая ветка. История исходных веток остается неизменной.

  ![git-merge](imgs/git-merge.png)

  *Преимущества*:
    1. Простота,
    2. Сохраняет полную историю и хронологический порядок,
    3. Поддерживает контекст ветки.

  *Недостатки*:
    1. История коммитов может быть заполнена (загрязнена) множеством коммитов,
    2. Отладка с использованием git bisect может стать сложнее.


- `git rebase` - поочерёдно переприменяет коммиты текущей ветки поверх целевой, создавая новые коммиты с новыми хешами (объединить их в один можно через интерактивный `git rebase -i`). В отличии от *merge*, *rebase* перезаписывает историю, потому что она передаётся завершенную работу из одной ветки в другую. В процессе устраняется нежелательная история.

  ![git-rebase](imgs/git-rebase.png)

  *Преимущества*:
    1. Упрощает потенциально сложную историю,
    2. Упрощение манипуляций с единственным коммитом,
    3. Избежание слияния коммитов в занятых репозиториях и ветках,
    4. Очищает промежуточные коммиты, делая их одним коммитом, что полезно для DevOps команд.

    *Недостатки*:
    1. Сжатие фич до нескольких коммитов может скрыть контекст
    2. Перемещение публичных репозиториев может быть опасным при     работе в команде,
    3. Появляется больше работы,
    4. Для восстановления с удаленными ветками требуется     принудительный пуш. Это приводит к обновлению всех веток, имеющих одно и то же имя, как локально, так и удаленно.

</details>

3. Чем `tag` отличается от `branch`?

<details>
  <summary>Ответ</summary>

И *tag* и *branch* представляют собой указатели на коммиты.
- Ветка представляет собой отдельный поток разработки, который может выполняться одновременно с другими разработками в той же кодовой базе. Коммит в ветке указывает на изменения, которые добавляются в новых коммитах
- Тег представляет собой версию определенной ветки в определенный момент времени.

*Tag* представляет собой версию той или иной ветки в определенный момент времени. *Branch* представляет собой отдельный поток разработки, который может выполнятся одновременно с другими разработками в той же кодовой базе.

</details>

4. В ветке *develop* есть коммит с изменениями, которые нужно перенести в ветку *master*. Как это сделать?

<details>
  <summary>Ответ</summary>

Необходимо найти хеш этого коммита и выполнить следующую комманду в ветке, в которую нужно перенести коммит.
```sh
git cherry-pick <commit_hash>
```

</details>

5. Для чего нужна команда `git commit --amend`?

<details>
  <summary>Ответ</summary>

`commit --amend` используется для исправления сообщения последнего коммита. Также возможно использовать, чтобы добавить файлы в индекс (`git add`), после добавить файлы в коммит `git commit --amend`.

</details>

6. Что такое Trunk-based development?

<details>
  <summary>Ответ</summary>

Trunk-based Development (TBD) - модель ветвления, в которой разработчики совместно работают над кодом в одной ветви, называемой "стволом" (trunk). При этом другие ветви имеют короткий срок жизни благодаря использованию документированных методов.

</details>

7. Состояние репозитория ушло на много коммитов вперед. Как откатить весь репозиторий к определенному коммиту?

<details>
  <summary>Ответ</summary>

git reset --hard <tag/branch/commit hash>

</details>

8. В репозиторий запушен коммит с изменениями в двух файлах. Как откатить изменения этого коммита?

<details>
  <summary>Ответ</summary>

git revert <commit hash>

</details>

9. Чем `git fetch` отличается от `git pull`?

<details>
  <summary>Ответ</summary>

- `git fetch` - скачивает новые коммиты и обновляет удалённые ветки (`origin/main`), **не трогая** локальные ветки и рабочий каталог. Безопасно, можно посмотреть изменения: `git log main..origin/main`.
- `git pull` = `git fetch` + интеграция в текущую ветку: по умолчанию `merge`, с `--rebase` - rebase.

Чтобы `pull` не создавал лишних merge-коммитов, настраивают поведение явно:
```sh
git config --global pull.rebase true    # или
git config --global pull.ff only        # только fast-forward, иначе ошибка
```

</details>

10. Что такое fast-forward merge? Чем он отличается от merge с merge-коммитом (`--no-ff`)?

<details>
  <summary>Ответ</summary>

Если целевая ветка не ушла вперёд с момента ответвления, Git при `merge` просто передвигает указатель ветки на последний коммит - это **fast-forward**, новый коммит не создаётся, история линейная.

Если обе ветки имеют новые коммиты, выполняется **трёхстороннее слияние** (three-way merge) с общим предком (`git merge-base`), создаётся merge-коммит с двумя родителями.
```sh
git merge --ff-only feature   # только fast-forward, иначе отказ
git merge --no-ff feature     # всегда создать merge-коммит (видно, что была ветка)
```
`--no-ff` сохраняет в истории границы фичи, `--ff-only` гарантирует линейную историю.

</details>

11. Как объединить несколько коммитов в один (squash)?

<details>
  <summary>Ответ</summary>

1. Интерактивный rebase последних N коммитов:
```sh
git rebase -i HEAD~3    # в редакторе заменить pick на squash (s) или fixup (f)
```
2. Не открывая редактор - мягкий reset и новый коммит:
```sh
git reset --soft HEAD~3 && git commit -m "feat: одна осмысленная фича"
```
3. Слить ветку одним коммитом: `git merge --squash feature && git commit`. То же делает кнопка "Squash and merge" в GitHub/GitLab.
4. Исправление старого коммита: `git commit --fixup=<sha>`, затем `git rebase -i --autosquash <base>`.

Squash переписывает историю - уже опубликованную ветку после этого нужно пушить с `--force-with-lease`.

</details>

12. Чем отличаются `git reset --soft`, `--mixed` и `--hard`? Когда использовать `reset`, а когда `revert`?

<details>
  <summary>Ответ</summary>

Все три переносят указатель текущей ветки (HEAD) на указанный коммит, отличаются тем, что делают с индексом и рабочим каталогом:
- `--soft` - индекс и файлы не меняются, изменения "отменённых" коммитов остаются подготовленными к коммиту;
- `--mixed` (по умолчанию) - индекс сбрасывается, изменения остаются в файлах как неподготовленные;
- `--hard` - индекс и рабочий каталог приводятся к коммиту, незакоммиченные изменения **теряются**.

`reset` переписывает историю - подходит для локальных, ещё не опубликованных коммитов. Для уже запушенных в общую ветку коммитов используют `git revert`: он создаёт новый коммит с обратными изменениями и не ломает историю коллегам. Revert merge-коммита требует указать родителя: `git revert -m 1 <merge_sha>`.

</details>

13. Сделали `git reset --hard` (или удалили ветку) и потеряли коммиты. Как их восстановить?

<details>
  <summary>Ответ</summary>

`git reflog` - локальный журнал перемещений HEAD и веток. Коммиты не удаляются сразу, пока на них ссылается reflog (по умолчанию записи хранятся 90 дней, для недостижимых коммитов - 30 дней, потом их может удалить `git gc`).
```sh
git reflog                         # найти нужное состояние, например HEAD@{3}
git reset --hard HEAD@{3}          # вернуть ветку
# или сохранить в новую ветку
git branch recovered <sha>
git reflog show feature            # журнал конкретной ветки
```
Если записи в reflog нет - можно поискать "висячие" коммиты: `git fsck --lost-found`. Незакоммиченные изменения, потерянные при `reset --hard`, reflog не вернёт (кроме файлов, добавленных в индекс - они есть как dangling blob).

</details>

14. Для чего нужен `git stash`? Чем `stash pop` отличается от `stash apply`?

<details>
  <summary>Ответ</summary>

`stash` временно убирает незакоммиченные изменения, возвращая рабочий каталог к чистому состоянию (например, чтобы срочно переключиться на hotfix).
```sh
git stash push -m "wip: форма логина"   # -u - включая неотслеживаемые файлы
git stash list
git stash show -p stash@{0}            # посмотреть содержимое
git stash apply stash@{0}              # применить и оставить в списке
git stash pop                          # применить и удалить из списка
git stash branch wip-login stash@{0}   # создать ветку из stash
git stash drop stash@{0}
```
При конфликте `pop` не удаляет запись из stash - её нужно удалить вручную после разрешения.

</details>

15. Как с помощью `git bisect` найти коммит, который сломал сборку?

<details>
  <summary>Ответ</summary>

`bisect` выполняет бинарный поиск по истории между "хорошим" и "плохим" коммитом: для 1000 коммитов достаточно ~10 проверок.
```sh
git bisect start
git bisect bad                 # текущий коммит сломан
git bisect good v1.4.0         # в этой версии всё работало
# Git переключается на середину; проверяем и отмечаем good/bad, пока не найдём коммит
git bisect reset               # вернуться туда, где начали
```
Автоматизация: `git bisect run ./test.sh` - скрипт возвращает 0, если коммит хороший, 1-127 (кроме 125) - плохой, 125 - пропустить (коммит нельзя проверить). Поэтому полезны атомарные, собирающиеся коммиты.

</details>

16. Как разрешить конфликт при merge или rebase?

<details>
  <summary>Ответ</summary>

1. `git status` - список конфликтующих файлов.
2. В файлах есть маркеры `<<<<<<<`, `=======`, `>>>>>>>` - оставить нужный вариант и удалить маркеры (или `git mergetool`).
3. `git add <file>` и продолжить: `git merge --continue` / `git rebase --continue` / `git cherry-pick --continue`.
4. Передумали - `git merge --abort` / `git rebase --abort`.

Взять версию одной стороны целиком: `git checkout --ours file` или `--theirs file`. Внимание: при **rebase** стороны меняются местами - `ours` это ветка, на которую переносим, `theirs` - ваши коммиты. `git config rerere.enabled true` запоминает разрешения и применяет их повторно. После разрешения - собрать и прогнать тесты.

</details>

17. Что такое detached HEAD и как не потерять сделанные в нём коммиты?

<details>
  <summary>Ответ</summary>

Обычно HEAD указывает на ветку (`ref: refs/heads/main`). Detached HEAD - HEAD указывает прямо на коммит: после `git checkout <sha>`, `git checkout v1.2.0`, во время rebase/bisect, в CI при checkout по SHA.

Коммитить можно, но они не принадлежат ни одной ветке: после переключения на другую ветку на них ничего не ссылается, и со временем `git gc` их удалит. Сохранить:
```sh
git switch -c hotfix-from-tag     # создать ветку из текущего состояния
# если уже ушли - найти SHA в git reflog и сделать git branch <name> <sha>
```

</details>

18. Чем `git push --force-with-lease` лучше `git push --force`?

<details>
  <summary>Ответ</summary>

После rebase/squash/amend опубликованной ветки нужен принудительный push. `--force` перезаписывает удалённую ветку безусловно - можно стереть коммиты коллеги, запушенные после вашего последнего fetch.

`--force-with-lease` перезаписывает ветку, только если на сервере она всё ещё указывает туда же, куда ваша `origin/<branch>`. Если кто-то успел запушить - push отклоняется.
```sh
git push --force-with-lease origin feature
git push --force-with-lease --force-if-includes origin feature   # защита от фонового fetch в IDE
```
Общие ветки (`main`, `release/*`) защищают на сервере (protected branches), запрещая force push полностью.

</details>

19. Файл добавлен в `.gitignore`, но Git продолжает его отслеживать. Почему и что делать?

<details>
  <summary>Ответ</summary>

`.gitignore` действует только на **неотслеживаемые** файлы. Если файл уже был закоммичен, правило игнорирования на него не влияет.
```sh
git rm --cached config/local.env      # убрать из индекса, файл на диске останется
git rm -r --cached .idea/
git commit -m "chore: stop tracking local config"
git check-ignore -v path/to/file      # какое правило (файл и строка) игнорирует путь
```
Другие места для правил: `.git/info/exclude` (локально, не коммитится) и глобальный `core.excludesFile` (например, для `.DS_Store`, файлов IDE). Файл при этом остаётся в истории - если это секрет, см. следующий вопрос.

</details>

20. В репозиторий закоммитили пароль/токен и запушили. Что делать?

<details>
  <summary>Ответ</summary>

1. **Сразу отозвать и перевыпустить секрет** - это главное. Секрет считается скомпрометированным: его могли забрать из клонов, форков, CI-логов, кэшей; очистка истории этого не отменяет.
2. Удалить из истории с помощью `git filter-repo` (рекомендуется вместо `git filter-branch`; альтернатива - BFG):
```sh
git clone git@example.com:team/app.git && cd app      # filter-repo требует свежий клон
git filter-repo --invert-paths --path config/secrets.env
# либо заменить только строку: git filter-repo --replace-text ../replacements.txt (строки вида токен==>REMOVED)
git remote add origin git@example.com:team/app.git    # filter-repo удаляет origin для защиты от случайного push
git push --force --all origin && git push --force --tags origin
```
3. Все хеши меняются: коллеги должны переклонировать репозиторий; у хостинга могут остаться ссылки на старые коммиты (PR, кэши) - обратиться в поддержку. Снять защиту веток на время force push.
4. Профилактика: pre-commit хуки и CI-сканеры (gitleaks, trufflehog), push protection / secret scanning на стороне GitHub/GitLab.

</details>

21. Чем lightweight-тег отличается от annotated? Как использовать теги для версионирования (SemVer)?

<details>
  <summary>Ответ</summary>

- **lightweight** (`git tag v1.2.0`) - просто указатель на коммит.
- **annotated** (`git tag -a v1.2.0 -m "Release 1.2.0"`) - отдельный объект с автором, датой, сообщением, может быть подписан (`-s`). Для релизов используют annotated.

Теги по умолчанию не пушатся:
```sh
git push origin v1.2.0
git push --follow-tags        # пушит annotated-теги, достижимые из пушимых коммитов
git describe --tags           # v1.2.0-5-g1a2b3c4: 5 коммитов после тега
git push origin --delete v1.2.0
```
**SemVer** `MAJOR.MINOR.PATCH`: MAJOR - несовместимые изменения API, MINOR - новая обратно совместимая функциональность, PATCH - исправления; пре-релизы `1.3.0-rc.1`. Опубликованный тег не переносят - выпускают новую версию.

</details>

22. Зачем подписывать коммиты и как это настроить?

<details>
  <summary>Ответ</summary>

Автор коммита (`user.name`, `user.email`) задаётся произвольно, подделать его тривиально. Подпись криптографически подтверждает, что коммит сделан владельцем ключа; GitHub/GitLab показывают "Verified", а правила защиты ветки могут требовать подписанные коммиты.

Проще всего - SSH-ключом (Git 2.34+), также поддерживаются GPG и X.509 (gitsign/Sigstore):
```sh
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgSign true
git log --show-signature -1       # для проверки SSH-подписей локально нужен gpg.ssh.allowedSignersFile
```
Публичный ключ нужно добавить в профиль хостинга как signing key.

</details>

23. Что такое Git hooks и фреймворк pre-commit? Можно ли на них полагаться для контроля качества?

<details>
  <summary>Ответ</summary>

Hooks - скрипты, которые Git вызывает на событиях: клиентские `pre-commit`, `commit-msg`, `pre-push`; серверные `pre-receive`, `update`, `post-receive`. Лежат в `.git/hooks` и **не версионируются**; общий каталог можно задать через `git config core.hooksPath .githooks`.

**pre-commit** - фреймворк для управления хуками через `.pre-commit-config.yaml` в репозитории (линтеры, форматтеры, `terraform fmt`, `shellcheck`, `gitleaks`, проверка больших файлов):
```sh
pip install pre-commit
pre-commit install            # установить хук в .git/hooks
pre-commit run --all-files    # прогнать по всем файлам (так же запускают в CI)
```
Клиентские хуки легко обойти (`git commit --no-verify`) или просто не установить, поэтому те же проверки обязательно дублируют в CI и через правила защиты веток/серверные хуки.

</details>

24. Чем `git submodule` отличается от `git subtree`?

<details>
  <summary>Ответ</summary>

- **submodule** - в родительском репозитории хранится ссылка на конкретный коммит другого репозитория (`.gitmodules` + gitlink). Код не копируется, история раздельная.
```sh
git submodule add https://github.com/org/lib.git vendor/lib
git clone --recurse-submodules <url>
git submodule update --init --recursive   # после обычного clone
git submodule update --remote             # подтянуть свежий коммит ветки
```
  Минусы: легко забыть инициализировать/обновить, detached HEAD внутри сабмодуля, нужен доступ ко всем репозиториям в CI.
- **subtree** - код другого репозитория копируется в подкаталог и становится частью истории: `git subtree add --prefix=vendor/lib https://github.com/org/lib.git main --squash`, обновление - `git subtree pull`. Клонирование прозрачное, но репозиторий растёт, а отправка изменений обратно (`git subtree push`) сложнее.

Для зависимостей кода часто лучше пакетный менеджер, чем любой из вариантов.

</details>

25. Монорепозиторий разросся, `clone` и `status` работают медленно. Какие средства Git помогают?

<details>
  <summary>Ответ</summary>

- **Shallow clone** - только последние коммиты (типично для CI): `git clone --depth 1 <url>`.
- **Partial clone** - вся история, но содержимое файлов докачивается по требованию: `git clone --filter=blob:none <url>`.
- **Sparse checkout** - в рабочий каталог выгружаются только нужные каталоги:
```sh
git sparse-checkout set services/api libs/common
```
- `git maintenance start` - фоновое обслуживание (commit-graph, prefetch, repack); `core.fsmonitor=true` и `core.untrackedCache=true` ускоряют `git status`; `scalar clone` включает всё это сразу.
- Большие бинарные файлы - в Git LFS.

Организационно для монорепо нужны CODEOWNERS, запуск CI только по изменённым путям и инструменты сборки с учётом графа зависимостей (Bazel, Nx, Turborepo).

</details>

26. Что такое Conventional Commits и зачем их использовать?

<details>
  <summary>Ответ</summary>

Соглашение о формате сообщений коммитов: `<type>(<scope>)!: <описание>`, например:
```
feat(auth): add OIDC login
fix(api): handle empty payload
refactor!: drop support for config v1

BREAKING CHANGE: config v1 is no longer supported
```
Типы: `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`. Связь с SemVer: `fix` -> PATCH, `feat` -> MINOR, `!` или `BREAKING CHANGE` -> MAJOR.

Зачем: читаемая история, автоматическая генерация CHANGELOG и версий (semantic-release, release-please), фильтрация коммитов. Формат проверяют хуком `commit-msg` (commitlint) и в CI - для PR-заголовков при squash merge.

</details>

27. Зачем появились `git switch` и `git restore`, если есть `git checkout`?

<details>
  <summary>Ответ</summary>

`git checkout` исторически выполнял две разные задачи - переключение веток и восстановление файлов, что приводило к ошибкам (например, `git checkout name` при совпадении имени ветки и файла). С Git 2.23 их разделили:
```sh
git switch feature              # переключиться на ветку
git switch -c feature           # создать и переключиться (аналог checkout -b)
git switch -                    # на предыдущую ветку
git switch --detach v1.2.0      # явно перейти в detached HEAD
git restore app.py              # отменить изменения файла в рабочем каталоге
git restore --staged app.py     # убрать из индекса (аналог reset HEAD app.py)
git restore --source=HEAD~2 app.py   # взять версию файла из другого коммита
```
`checkout` продолжает работать, но в скриптах и обучении предпочтительнее новые команды.

</details>