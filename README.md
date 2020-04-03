# 1. 建立openldap

## 說明

為了降低使用者註冊負擔，及未來其他docker服務應用，建立ldap做為人事系統使用。
除了ldap系統，也使用phpldapadmin來加速管理，可以透過gui介面來修改人事資料。因此yml檔會有兩段，一個是openldap，一個是phpldapadmin。
port的部分因為8181被部內wiki使用，因此都使用82開頭的port。

## 流程

將對應的docker環境建置好，並且安裝docker-compose，完成後cd到ldap/docker-compose.yml路徑下，用以下語法即可。

```shell
docker-compose up
```

完成後瀏覽器打上phpldapadmin的url+port，就可以進去ldap的管理畫面。
預設帳號是cn=admin,dc=example,dc=org，密碼是admin。

# 2. 建立gitlab

## 說明

gitlab建立好後，要進去container裡面修改gitlab.rb，設定與ldap的連線，才能使用ldap的方式登入。

## 流程

將對應的docker環境建置好，並且安裝docker-compose，完成後cd到外面docker-compose.yml路徑下，下以下語法即可。

```shell
docker-compose up
```

進去container裡面，$字號後面請自行帶入container的編號

```shell
docker exec -it $containerID bash
```

到gitlab.rb進行設定修改。

```shell
vim /etc/gitlab/gitlab.rb
```

原則上可以整段貼上，點i開始編寫，貼上後ESC，然後wq存檔離開。


```rb
gitlab_rails['ldap_enabled'] = true

##! **remember to close this block with 'EOS' below**
gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
  main: # 'main' is the GitLab 'provider ID' of this LDAP server
    label: 'LDAP'
    host: '35.194.203.246' #部門VM的IP
    port: 8200 #openladp的port
    uid: 'uid'
    bind_dn: 'cn=admin,dc=example,dc=org' #openladp的帳號
    password: 'admin' #openladp的密碼
    encryption: 'plain' # "start_tls" or "simple_tls" or "plain"
    verify_certificates: false
    active_directory: true
    allow_username_or_email_login: false
    lowercase_usernames: false
    block_auto_created_users: false
    base: 'ou=user,dc=example,dc=org' #openladp的用戶來源
    user_filter: ''
    ## EE only
    group_base: ''
    admin_group: ''
    sync_ssh_keys: false

EOS
```

完成後重起gitlab設定。

```shell
gitlab-ctl reconfigure
```

可以用以下語法確認有無連接ldap成功。

```shell
gitlab-rake gitlab:ldap:check
```

好了以後瀏覽器打上gitlab網址就能使用了，第一次要打上root的密碼，為主要管理員帳號，之後就可以用一般人ldap帳號登入。
一到香噴噴的部內gitlab就建置完成了。

如果要將HTTP URL的內容修正確，到gitlab.rb進行設定修改。
加上下面兩句

```shell
external_url 'http://ip:8080'
nginx['listen_port'] = 80
```



