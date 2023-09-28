# PG BootCamp Russia 2023

**Доклад**: `Билдь или не билдь...Или Как достойно собрать PostgreSQL из исходников`\
**Краткая аннотация**: Мастер-класс про то как собрать postgresql из исходников и  протестировать собранное. Как работать с патчами к коду postgres. И зачем это все может быть нужно обычному айтишнику.\
**Спикер**: Вадим Пономарев, Tantor Labs, ГК «Астра» (https://t.me/vbponomarev)<br>
**Презентация**: [ссылка](https://docs.google.com/presentation/d/1Y8reNmlylI9s2TvBDErEwdWkel9LDhfq/edit?usp=sharing&ouid=110533282330244212380&rtpof=true&sd=true)<br>

# Tutorial
Сборка только под Linux и на Linux, конкретно в примерах - Ubuntu 22.04 на WSL и AstraLinux 1.7 в docker. Но лучше, конечно, в контейнере.

## Запуск стенда
 - Ubuntu 22.04 на windows, в WSL: `wsl --install -d Ubuntu-22.04`
 - Ubuntu 22.04 в docker: `sudo docker run --rm -it ubuntu:22.04 bash`
 - ALSE 1.7 в docker: `sudo docker run --rm -it registry.astralinux.ru/library/alse:1.7.4 bash`

## Подготовка (Ubuntu 22.04)
 - ставим dev-пакеты и зависимости
```
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update
sudo apt-get install -y \
	git tree nano \
	build-essential cpanminus slapd ldap-utils libldap2-dev \
	autoconf bison clang-11 devscripts dpkg-dev flex \
	libldap2-dev libdbi-perl libgssapi-krb5-2 libicu-dev \
	krb5-kdc krb5-admin-server libssl-dev libpam0g-dev \
	libkrb5-dev krb5-user libcurl4-openssl-dev \
	perl perl-modules libipc-run-perl libtest-simple-perl libtime-hires-perl \
	liblz4-dev libpam-dev libreadline-dev libselinux1-dev libsystemd-dev \
	libxml2-dev libxslt-dev libzstd-dev llvm-11-dev locales-all pkg-config \
	python3-dev uuid-dev zlib1g-dev \
    openjade docbook-xml docbook-xsl opensp libxml2-utils xsltproc \
    libjson-perl curl time
```

 - опционально, если будем использовать TAP-тесты:
```
sudo cpanm TAP::Harness::Archive TAP::Parser::SourceHandler::pgTAP IPC::Run
```
 - загрузка исходных кодов:
```
git clone https://github.com/postgres/postgres 
cd postgres
git status
git branch --all
```

## Подготовка (ALSE 1.7)
ВНИМАНИЕ! Надеюсь, вы понимаете, что собранный таким образом Postgres не будет иметь поддержки мандатного доступа, т.е. ставить его имеет смысл только на вариант дистрибутива ALSE Орел. 
 - ставим dev-пакеты и зависимости
```
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y \
    jq git tree nano wget python3-minimal systemd vim procps \
    build-essential slapd ldap-utils libldap2-dev \
    autoconf bison clang-11 devscripts dpkg-dev flex \
    libldap2-dev libdbi-perl libgssapi-krb5-2 libicu-dev \
    krb5-kdc krb5-admin-server libssl-dev libpam0g-dev \
    libkrb5-dev krb5-user libcurl4-openssl-dev \
    perl perl-modules libipc-run-perl libtest-simple-perl libtime-hires-perl \
    liblz4-dev libpam-dev libreadline-dev libselinux1-dev libsystemd-dev \
    libxml2-dev libxslt-dev libzstd-dev llvm-11-dev locales-all pkg-config \
    python3-dev uuid-dev zlib1g-dev \
    openjade docbook-xml docbook-xsl opensp libxml2-utils xsltproc \
    libjson-perl curl time
```

 - опционально, если будем использовать TAP-тесты:
```
apt-get install libmodule-install-perl
cpan TAP::Harness::Archive TAP::Parser::SourceHandler::pgTAP IPC::Run
```
 - загрузка исходных кодов:
```
git clone https://github.com/postgres/postgres 
cd postgres
git status
git branch --all
```

## Сборка и тестирование
 - конфигурируем и собираем
```
./configure --help

export LLVM_CONFIG=/usr/bin/llvm-config-11
export CLANG=/usr/bin/clang-11
export WORKSPACE_PATH=/home/vponomarev

./configure --prefix=$WORKSPACE_PATH/postgres_bin --enable-debug --with-python\
	--with-llvm --with-lz4 --with-zstd --with-ssl=openssl --enable-nls='ru' CFLAGS='-O2 -pipe' --enable-tap-tests
make world-bin -j $(nproc)
```
 - гоняем тесты
```
make check-world -j $(nproc)
```
 - если включали TAP-тесты, то запускаем и их тоже
```
make -C src/bin check
```
 - устанавливаем собранное, создаем кластер баз
```
make install-world-bin
PATH=$WORKSPACE_PATH/postgres_bin/bin:$PATH
LD_LIBRARY_PATH=$WORKSPACE_PATH/postgres_bin/lib
PGDATA=$WORKSPACE_PATH/postgres_data
mkdir -p $PGDATA
initdb -D $PGDATA
```
 - настраиваем coredump
```
ulimit -S -c unlimited
sudo sysctl -w kernel.core_pattern=/var/crash/core-%e-%s-%u-%g-%p-%t
ulimit -a
```
 - для экономии места можно отключить сохранение в coredump shared memory процесса, если не надо сохранять содержимое shared_buffers
```
echo 0x0021 > /proc/self/coredump_filter
```
 - запускаем и гоняем тесты на созданной/запущенной базе, смотрим результаты
```
pg_ctl -D $PGDATA start
make installcheck-world
less ./src/test/recovery/tmp_check/log/regress_log*
```

## Дебаг и применение патчей
 - провоцируем stack overflow, исследуем coredump
```
(n=1000000; printf "BEGIN;"; for ((i=1;i<=$n;i++)); do printf "SAVEPOINT s$i;"; done;\
	printf "SELECT pg_log_backend_memory_contexts(pg_backend_pid())") | psql postgres > /dev/null

gdb "$WORKSPACE_PATH/postgres_bin/bin/postgres" /var/crash/core-postgres-* --ex 'bt full' --batch | less

pg_ctl -D $PGDATA stop
make distclean
```
 - находим похожую ошибку тут: https://commitfest.postgresql.org/44/4239/
 - скачиваем, адаптируем и применяем патч:
```
wget https://www.postgresql.org/message-id/attachment/147759/v1-0001-Add-some-checks-to-avoid-stack-overflow.patch
git apply -v ./v1-0001-Add-some-checks-to-avoid-stack-overflow.patch
```
 - не применилось, убираем лишее адаптируем, применяем снова
...
 - собираем, ставим, тестируем:
```
./configure --prefix=$WORKSPACE_PATH/postgres_bin --enable-debug --with-python\
	--with-llvm --with-lz4 --with-zstd --with-ssl=openssl --enable-nls='ru' CFLAGS='-O2 -pipe'
make world-bin -j $(nproc)
make install-world-bin
pg_ctl -D $PGDATA start
(n=1000000; printf "BEGIN;"; for ((i=1;i<=$n;i++)); do printf "SAVEPOINT s$i;"; done;\
	printf "SELECT pg_log_backend_memory_contexts(pg_backend_pid())") | psql postgres > /dev/null
```
