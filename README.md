# Сборка NGINX 1.30.2 для BitrixVM CentOS 7 с дополнительными модулями

Инструкция по сборке NGINX 1.30.2 для использования на виртуальной машине BitrixVM со всеми необходимыми модулями (push-stream, headers-more, mod_zip, brotli).

## ⚠️ Важное предупреждение

- Инструкция предназначена для **изолированной виртуальной машины BitrixVM**
- Перед началом работы **обязательно** сделайте полный снапшот ВМ или резервную копию через `bx-backup`
- Все команды выполняются от пользователя `root`
- Пере началом проверьте версию openssl( openssl version) , если стоит 1.0.* то установку начинаем по второму варианту с обновленной openssl

---

## 📋 Пошаговая инструкция

### Шаг 1. Подготовка окружения и установка инструментов

Установите базовые пакеты для компиляции и Git:
```
yum install -y git gcc gcc-c++ make wget tar gzip perl-ExtUtils-Embed
```
Шаг 2. Создание рабочей директории
```
mkdir -p ~/nginx_build && cd ~/nginx_build
```
Шаг 3. Установка зависимостей
Установите системные библиотеки, включая Brotli для сжатия:
```
yum install -y openssl-devel pcre-devel zlib-devel brotli brotli-devel
```
Для CentOS 7: Если пакеты brotli и brotli-devel не найдены, подключите EPEL репозиторий:
```
yum install -y epel-release
yum install -y brotli brotli-devel
```
Шаг 4. Загрузка исходного кода NGINX 1.30.2
```
cd ~/nginx_build
wget https://nginx.org/download/nginx-1.30.2.tar.gz
tar -xzf nginx-1.30.2.tar.gz
```
Шаг 5. Клонирование внешних модулей
Эти модули необходимы для полной совместимости с BitrixVM:
```
cd ~/nginx_build
```
# 1. push-stream-module (для веб-сокетов и стриминга)
```
git clone https://github.com/wandenberg/nginx-push-stream-module.git
```
# 2. headers-more (для управления заголовками)
```
git clone https://github.com/openresty/headers-more-nginx-module.git
```
# 3. mod_zip (для динамической архивации)
```
git clone https://github.com/evanmiller/mod_zip.git
```
# 4. ngx_brotli (сжатие Brotli)
```
git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli && git submodule update --init && cd ..
```
Примечание: Модули mod_zip и push-stream могут давно не обновляться, но они критичны для работы некоторых функций Bitrix. Если клонирование не удаётся, проверьте наличие форков или временно отключите соответствующий --add-module.

Шаг 6. Конфигурация сборки
Перейдите в директорию исходников и запустите configure с параметрами, приближенными к родным для BitrixVM:

```
cd ~/nginx_build/nginx-1.30.2

./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --http-client-body-temp-path=/var/cache/nginx/client_temp \
    --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
    --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
    --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
    --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
    --user=nginx --group=nginx \
    --with-openssl-opt=enable-tls1_3 \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-http_auth_request_module \
    --with-http_v2_module \
    --with-mail --with-mail_ssl_module \
    --with-file-aio \
    --add-module=../nginx-push-stream-module \
    --add-module=../mod_zip \
    --add-module=../headers-more-nginx-module \
    --add-module=../ngx_brotli \
    --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic'
```
Шаг 7. Компиляция и установка
Если configure прошёл без ошибок, запустите сборку и установку:
```
make && make install
```
Процесс может занять 5-10 минут в зависимости от мощности ВМ.

Шаг 8. Проверка конфигурации и версии
```
/usr/sbin/nginx -t
/usr/sbin/nginx -v
```
Вы должны увидеть:
```
nginx version: nginx/1.30.2

syntax is ok
```
Шаг 9. Перезапуск NGINX и очистка
```
systemctl restart nginx
```


# Проверьте, что сервер работает
```
systemctl status nginx
```
# Удалите временную папку сборки (опционально)
```
rm -rf ~/nginx_build
```
Шаг 10. Проверка загруженных модулей
Убедитесь, что все модули активны:
```
/usr/sbin/nginx -V 2>&1 | grep --color 'modules'
```
В выводе должны быть видны пути к ngx_brotli, push_stream_module и другим модулям.

🔄 Устранение ошибок при configure
Ошибка	Решение
brotli not found	Установите brotli-devel (см. Шаг 3)
push-stream module error	Временно удалите --add-module=../nginx-push-stream-module из строки configure
OpenSSL version mismatch	Установите openssl11-devel или используйте --with-openssl=../openssl-3.0.x (скачав исходники OpenSSL отдельно)
libbrotlicommon.so.1() not found	Установите EPEL и brotli-devel, либо соберите brotli из исходников
⚠️ Важные замечания после сборки
Обновление через штатные средства более невозможно. Отныне вы обновляете NGINX только вручную, повторяя эту процедуру.

При обновлении BitrixVM (через bx-update) ваш самособранный NGINX может быть заменён на стандартный — будьте готовы повторить сборку.

Данная инструкция не затрагивает модуль PageSpeed (если он использовался), так как он не входит в стандартную поставку BitrixVM.

🛡️ Проверка безопасности
После сборки рекомендуется проверить, что уязвимость CVE-2026-42945 (NGINX Rift) устранена:


# Проверьте версию (должна быть 1.30.2 или выше)
```
nginx -v
```
# Сборка NGINX 1.30.2 для BitrixVM с дополнительными модулями , для тех кто решил скомпилировать с обновлённой openssl
Устанавливаем пакеты 
```
yum groupinstall 'Development Tools' -y
yum install perl-core zlib-devel wget pcre-devel -y
```
Скачиваем openssl
```
cd /usr/src
wget https://github.com/openssl/openssl/releases/download/openssl-3.5.6/openssl-3.5.6.tar.gz
tar -zxf openssl-3.5.6.tar.gz
cd openssl-3.5.6
```
### Шаг 1. Подготовка окружения и установка инструментов

Установите базовые пакеты для компиляции и Git:
```
yum install -y git gcc gcc-c++ make wget tar gzip perl-ExtUtils-Embed
```
Шаг 2. Создание рабочей директории
```
mkdir -p ~/nginx_build && cd ~/nginx_build
```
Шаг 3. Установка зависимостей
Установите системные библиотеки, включая Brotli для сжатия:
```
yum install -y openssl-devel pcre-devel zlib-devel brotli brotli-devel
```
Для CentOS 7: Если пакеты brotli и brotli-devel не найдены, подключите EPEL репозиторий:
```
yum install -y epel-release
yum install -y brotli brotli-devel
```
Шаг 4. Загрузка исходного кода NGINX 1.30.2
```
cd ~/nginx_build
wget https://nginx.org/download/nginx-1.30.2.tar.gz
tar -xzf nginx-1.30.2.tar.gz
```
Шаг 5. Клонирование внешних модулей
Эти модули необходимы для полной совместимости с BitrixVM:
```
cd ~/nginx_build
```
# 1. push-stream-module (для веб-сокетов и стриминга)
```
git clone https://github.com/wandenberg/nginx-push-stream-module.git
```
# 2. headers-more (для управления заголовками)
```
git clone https://github.com/openresty/headers-more-nginx-module.git
```
# 3. mod_zip (для динамической архивации)
```
git clone https://github.com/evanmiller/mod_zip.git
```
# 4. ngx_brotli (сжатие Brotli)
```
git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli && git submodule update --init && cd ..
```
Примечание: Модули mod_zip и push-stream могут давно не обновляться, но они критичны для работы некоторых функций Bitrix. Если клонирование не удаётся, проверьте наличие форков или временно отключите соответствующий --add-module.

Шаг 6. Конфигурация сборки
Перейдите в директорию исходников и запустите configure с параметрами, приближенными к родным для BitrixVM:


Тут можно указать у ./configure параметр   --with-openssl=/usr/src/openssl-3.5.6 и nginx установиться с обновленной openssl
```
cd ~/nginx_build/nginx-1.30.2

./configure \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --http-client-body-temp-path=/var/cache/nginx/client_temp \
    --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
    --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
    --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
    --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
    --user=nginx --group=nginx \
    --with-openssl-opt=enable-tls1_3 \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-http_auth_request_module \
    --with-http_v2_module \
    --with-mail --with-mail_ssl_module \
    --with-file-aio \
    --add-module=../nginx-push-stream-module \
    --add-module=../mod_zip \
    --with-openssl=/usr/src/openssl-3.5.6 \
    --add-module=../headers-more-nginx-module \
    --add-module=../ngx_brotli \
    --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic'
```
Компиляция и установка.
Если configure прошёл без ошибок, запустите сборку и установку:
```
make && make install
```

Для тех кто решил установить и в систему 
```
cd /usr/src/openssl-3.5.6
./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
make
make test
make install
```
Обновляем пути
```
echo "/usr/local/ssl/lib64" | sudo tee /etc/ld.so.conf.d/openssl.conf
ldconfig -v
echo "export PATH=/usr/local/ssl/bin:\$PATH" | sudo tee /etc/profile.d/openssl.sh
source /etc/profile.d/openssl.sh
```
Проверяем 
```
openssl version
```
Если ошибки
```

nginx: [emerg] SSL_CTX_use_certificate("/etc/nginx/ssl/pool_manager.pem") failed (SSL: error:0A00018F:SSL routines::ee key too small)
nginx: [emerg] SSL_CTX_use_certificate("/etc/nginx/ssl/cert.pem") failed (SSL: error:0A00018F:SSL routines::ee key too small) 

```
Делаем новые сертификаты
```
openssl req -x509 -newkey rsa:2048 -keyout /etc/nginx/ssl/cert_new.key -out /etc/nginx/ssl/cert_new.crt -days 3656660 -nodes

cat /etc/nginx/ssl/cert_new.crt /etc/nginx/ssl/cert_new.key > /etc/nginx/ssl/cert.pem
cat  /etc/nginx/ssl/cert_new.key >> /etc/nginx/ssl/cert.pem
```
```
openssl req -x509 -newkey rsa:2048 -keyout /etc/nginx/ssl/pool_manager_new.key -out /etc/nginx/ssl/pool_manager_new.crt -days 3656660 -nodes

cat /etc/nginx/ssl/pool_manager_new.crt /etc/nginx/ssl/pool_manager_new.key > /etc/nginx/ssl/pool_manager.pem
cat /etc/nginx/ssl/pool_manager_new.key >> /etc/nginx/ssl/pool_manager.pem
```
