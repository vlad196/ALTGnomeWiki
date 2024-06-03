# Сборка RPM-пакетов

***Статья составлена с учётом наставлений Hihin Ruslan (ALT Linux Team)***

:::info
Для понимания сразу оговорюсь

***$*** - выполнение команды от обычного пользователя

***#*** - выполнения команды от пользователя root
:::

## Клонирование репозитория и начало сборки

Для начала создаём каталог _(если не создан)_ где мы будем хранить клон репозиториев, пускай это будет bild_packages

```
$ mkdir ~/bild_packages
```

Далее выполняем вход в каталог и выполняем клонирование.

```
$ cd bild_packages/
$ git clone https://gitlab.gnome.org/subpop/damask.git 
```

_В случае использования автоматического создания спек файла, клонирование репозитория будет сделано (или не сделано) при создании спек файла. см. раздел (Spec)_

После чего мы заходим в клонированный репозиторий и создаём директорию **.gear** где размещаем файл правил (rules) и файл правил сборки (spec)

```
$ cd damask/
$ mkdir .gear
$ cd .gear/
$ touch rules damask.spec
```

Далее пишем наш rules и damask.spec

После чего выходим в каталог выше и добавляем файлы к репозиторию, чтобы при сборке они были видны

```
$ cd ..
$ git add .gear/rules .gear/damask.spec
```

Далее запускаем сборку

```
$ gear-hsh --no-sisyphus-check --commit -v
```

## Vendoring - "вендоринг"

Некоторые пакеты (к примеру RUST, GO, Node или Java) могут требовать дополнительных модулей, которые скачиваются во время сборки, однако при сборке доступ к интернету запрещён по умолчанию.

Для решения данной проблемы необходимо дать доступ хешеру к сети Интернет и выкачать недостающие модули зайдя во внутрь хешера.

:::info
**ВАЖНО**
Прежде чем зайти в хешер, нужно его собрать.
Хешер собирается командой сборки.
То есть если туда зайти сразу, он будет пустой, если запустить команду сборки пакета, он соберётся и после входа во внутрь хешера он будет не пустой.
Если запустить сборку другого пакета, внутри хешера будет другой пакет.
:::

Для этого сначала посмотрим наш сервер DNS

```
# cat /etc/resolv.conf
```

Далее пробрасываем его в хешер, это нужно для того, что бы хешер смог увидеть Интернет.

```
$ hsh-shell --rooter
echo nameserver 8.8.8.8 >> /etc/resolv.conf
exit
```

Где 8.8.8.8 = вашему DNS, который вы увидели командой выше.

Для удобства добавляем в наш хешер файловый менеджер MC, также в примере ниже указан принцип как добавить во внутрь хешера нужные пакеты.

```
$ hsh-install mc
```

Заходим в наш хешер с доступом в сеть Интернет.

```
$ share_ipc=yes share_network=yes hsh-shell --mount=/dev/pts,/proc
```

После успешного входа, как вариант, открываем MC и переходим в каталог пакета

```
/usr/src/RPM/BUILD/НАШ_ПАКЕТ
```

В данном каталоге будут лежать файлы отвечающие _(назовём это так)_ за дополнительные модули, где мы выполняем команду на скачивание этих модулей

**В случае с:**

### Rust

Это будет файл **Cargo.toml** , по этому нужно выполнить команду

```
cargo vendor
```

Тем самым будет создан каталог **vendor** с модулями

Также в случае с Rust необходимо прописать файл конфига, чтобы сборщик видел от куда брать модули.

Создаём каталог **.cargo** и добавляем файл **config.toml** где будут прописаны связи, чтобы при сборке билдер понимал, что ему нужно брать файлы не с сети Интернет, а с каталога vendor.

```
$ mkdir .cargo
$ cd .cargo
$ touch config.toml
```

Информация по заполнению **config.toml** указана в терминале во время выполнения команды _carogo vendor_

Обычно это

```
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

Далее добавляем файл config.toml в git

```
$ git add .cargo/config.toml
```

### Go

Выполняем команду

```
go mod vendor
```

Больше никаких телодвижений не требуеся

### Node

Выполняем команду

```
yarn install
```

Тут стоит обратить внимание, что перед использованием **yarn** необходимо установить его во внутрь хереша

После выполнения будет создан каталог с модулями **node_modules**

### Java

Рекомендую воспользоваться инструкцией с 

https://www.altlinux.org/Packaging/Vendoring


***

:::info
Обращаю внимание, что ниже будет описана работа сборки вне хешера, после скачивания доп модулей.
:::

После того, как были скачены дополнительные модули их необходимо переместить хешера в наш каталог с репозиторием, к примеру

```
От сюда
$ ~/hasher/chroot/usr/scr/RPM/BUILD/НАШ_ПАКЕТ/

Сюда
$ ~/build_packages/НАШ_ПАКЕТ

То есть на выходе будет путь
$ ~/build_packages/НАШ_ПАКЕТ/vendor

в случае с node
$ ~/build_packages/НАШ_ПАКЕТ/node_modules
```

:::info
Обращаю внимание, что далее приводится пример по добавлению каталога с доп модулями в git, без создания дополнительного vendor.tar
Этого вполне достаточно, хотя иногда бывают ситуации когда нужен отдельный tar, но обычно добавления в git, как уже сказал - достаточно.
:::

Для обработки данных модулей нам необходимо добавить в git непосредственно весь каталог с модулями.

```
git add --all -f КАТАЛОГ_С_МОДУЛЯМИ
```

На практике возможно ситуация, что конструкция **git add .** не сработает, по этому используется конструкция выше.

Также стоит обратить внимание, что для Rust необходимо прописать **config.toml** как указано выше.

Далее просто запускаем сборку

```
$ gear-hsh --no-sisyphus-check --commit -v
```

## Сборка внутри хешера

Сборка внутри хешера осуществятся по средствам **rpm**

После того, как хешер собран и осуществлён вход можно приступить к сборке и экспериментам внутри хешера.

Для этого необходимо перейти в каталог со spec файлом и просо запустить сборку

```
cd /usr/src/RPM/SPECS/
rpm -ba НАШ_СПЕК.spec
```