# RPM
# Управление пакетами. Дистрибьюция софта

# 1) создать свой RPM

заходми под супер юзером
sudo su
и необходимо выполнитьустановку всех необходимых пакетов:

{ yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils gcc

Скачиваем SPRM паакет NGINX для дальнейшей работы над ним

wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.18.0-1.el7.ngx.src.rpm

Надо также скачать и разархивировать openssl

wget https://www.openssl.org/source/latest.tar.gz

tar -xvf latest.tar.gz

Заранее поставим все зависимости чтобы в процессе сборки не было ошибок

yum-builddep rpmbuild/SPECS/nginx.spec


Воспользуемся rpm -i

rpm -i nginx-1.18.0-1.el7.ngx.src.rpm

Переходим в каталог rpmbuild:

ls -nh
нас интересует вот этот путь 

drwxr-xr-x. 2 0 0 24 Sep 16 19:33 SPECS


В папке SPECS лежит spec-файл, который описывает что и как собирать, нам нужно поправить SPECS/nginx.spec:

%build ./configure %{BASE_CONFIGURE_ARGS}
--with-cc-opt="%{WITH_CC_OPT}"
--with-ld-opt="%{WITH_LD_OPT}"
--with-openssl=/root/rpmbuild/openssl-1.1.1g make %{?_smp_mflags} %{__mv} %{bdir}/objs/nginx
 %{bdir}/objs/nginx-debug ./configure %{BASE_CONFIGURE_ARGS}
--with-cc-opt="%{WITH_CC_OPT}"
--with-ld-opt="%{WITH_LD_OPT}" make %{?_smp_mflags}

можем собирать пакет - rpmbuild -bb SPECS/nginx.spec

Видим два собранных пакета:

[root@otuslinux rpmbuild]# ls -l /root/rpmbuild/RPMS/x86_64/
total 2524
-rw-r--r--. 1 root root 789552 Sep 16 19:50 nginx-1.18.0-1.el7.ngx.x86_64.rpm
-rw-r--r--. 1 root root 1792396 Sep 16 19:50 nginx-debuginfo-1.18.0-1.el7.ngx.x86_64.rpm

Устанавливаем свой rpm пакет

yum localinstall -y RPMS/x86_64/nginx-1.18.0-1.el7.ngx.x86_64.rpm

Запускаем nginx

systemctl start nginx 
systemctl status nginx


# 2) создать свой репо

Первым делом создаем папку нашего nginx - > 

mkdir /usr/share/nginx/html/repo

Копируем туда наш собранный RPM и, например, RPM для установки репозитория Percona-Server

cp rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm/usr/share/nginx/html/repo/

wget https://www.percona.com/redir/downloads/percona-release/redhat/1.0-21/percona-release-1.0-21.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-1.0-21.noarch.rpm

Создаем репозиторий -
Инициализируем репозиторий командой

createrepo /usr/share/nginx/html/repo/ 

В location / в файле /etc/nginx/conf.d/default.conf добавим директиву autoindex on. В результате location будет выглядеть так:

location / { root /usr/share/nginx/html; index index.html index.htm; autoindex on;

Проверяем синтаксис 

nginx -t 

и делаем 

nginx -s reload

Теперь можем просмотреть наши пакеты в браузере HTTP  или выполинить 

curl -a http://localhost/repo/

... nginx-1.18.0-1.el7.ngx.x86_64.rpm 16-Sep-2020 20:19 789552
percona-release-1.0-21.noarch.rpm 07-Jul-2020 10:33 17956

Чтобы создать свой репозиторий

создаем файл /etc/yum.repos.d/otus.repo и добавляем в него следующее:

[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1

Убедимся что репозиторий подключился и посмотрим что в нем есть:

yum repolist enabled | grep otus 

yum list --showduplicates | grep otus
