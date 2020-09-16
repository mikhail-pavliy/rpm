# RPM
# Управление пакетами. Дистрибьюция софта
1) создать свой RPM

заходми под супер юзером
sudo su
и необходимо выполнитьустановку всех необходимых пакетов:

yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils openssl-devel zlib-devel pcre-devel gcc

Скачиваем SPRM паакет NGINX для дальнейшей работы над ним

wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.18.0-1.el7.ngx.src.rpm

Воспользуемся rpm -i

rpm -i nginx-1.18.0-1.el7.ngx.src.rpm

Переходим в каталог /root/rpmbuild:

ls -nh
total 8.0K
drwxr-xr-x. 3 0 0 26 Aug 19 12:54 BUILD
drwxr-xr-x. 2 0 0 6 Aug 19 13:19 BUILDROOT
drwxr-xr-x. 3 0 0 20 Aug 19 13:19 RPMS
drwxr-xr-x. 2 0 0 4.0K Aug 19 12:53 SOURCES
drwxr-xr-x. 2 0 0 24 Aug 19 12:53 SPECS
drwxr-xr-x. 2 0 0 6 Aug 19 12:54 SRPMS

В папке SPECS лежит spec-файл, который описывает что и как собирать, внесем изменения в SPECS/nginx.spec ,добавив в секцию %build необходимый нам модуль OpenSSL:

%build ./configure %{BASE_CONFIGURE_ARGS}
--with-cc-opt="%{WITH_CC_OPT}"
--with-ld-opt="%{WITH_LD_OPT}"
--with-openssl=/root/rpmbuild/openssl-1.1.1f make %{?_smp_mflags} %{__mv} %{bdir}/objs/nginx
%{bdir}/objs/nginx-debug ./configure %{BASE_CONFIGURE_ARGS}
--with-cc-opt="%{WITH_CC_OPT}"
--with-ld-opt="%{WITH_LD_OPT}" make %{?_smp_mflags}

Установим зависимости - yum-builddep SPECS/nginx.spec, затем собираем - rpmbuild -bb SPECS/nginx.spec
Видим два собранных пакета:

[root@otuslinux rpmbuild]# ls -l /root/rpmbuild/RPMS/x86_64/
total 2524
-rw-r--r--. 1 root root 789552 Aug 19 13:19 nginx-1.18.0-1.el7.ngx.x86_64.rpm
-rw-r--r--. 1 root root 1792396 Aug 19 13:19 nginx-debuginfo-1.18.0-1.el7.ngx.x86_64.rpm

Устанавливаем rpm пакет

yum localinstall -y RPMS/x86_64/nginx-1.18.0-1.el7.ngx.x86_64.rpm

Запускаем nginx - systemctl start nginx и можем посмотреть с какими параметрами nginx был скомпилирован, выполнив nginx -V

root@otuslinux rpmbuild]# nginx -V
nginx version: nginx/1.18.0 built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) built with OpenSSL 1.0.2k-fips 26 Jan 2017 TLS SNI support enabled configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-.... ... -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'

Можно было бы собрать nginx через make&&make install, но это некошерно и могут побить по рукам )

Создадим свой репозиторий
Для начала создаем папку в / нашего nginx - > mkdir /usr/share/nginx/html/repo
Копируем наш скомпилированный пакет nginx в папку с будущим репозиторием и скачиваем дополнительно пакет

wget https://www.percona.com/redir/downloads/percona-release/redhat/1.0-21/percona-release-1.0-21.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-1.0-21.noarch.rpm

Создаем репозиторий - createrepo /usr/share/nginx/html/repo/ и createrepo --update /usr/share/nginx/html/repo/

В location / в файле /etc/nginx/conf.d/default.conf добавим директиву autoindex on. В результате location будет выглядеть так:

location / { root /usr/share/nginx/html; index index.html index.htm; autoindex on;

Проверяем синтаксис nginx -t и делаем nginx -s reload

[root@otuslinux rpmbuild]# nginx -t nginx: the configuration file /etc/nginx/nginx.conf syntax is ok nginx: configuration file /etc/nginx/nginx.conf test is successful

Теперь можем просмотреть наши пакеты через HTTP curl -a http://localhost/repo/

[root@otuslinux rpmbuild]# curl -a http://localhost/repo/
... nginx-1.18.0-1.el7.ngx.x86_64.rpm 19-Aug-2020 13:19 789552
percona-release-1.0-21.noarch.rpm 07-Jul-2020 10:33 17956

Чтобы протестировать репозиторий - создаем файл /etc/yum.repos.d/otus.repo и вписываем в него следующее:

[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1

Проверим подключенный репозиторий - yum repolist enabled | grep otus или yum list --showduplicates | grep otus

[root@otuslinux rpmbuild]# yum repolist enabled | grep otus
Failed to set locale, defaulting to C
otus otus-linux

[root@otuslinux rpmbuild]# yum list --showduplicates | grep otus
Failed to set locale, defaulting to C
nginx.x86_64 1:1.18.0-1.el7.ngx otus
percona-release.noarch 1.0-21 otus
