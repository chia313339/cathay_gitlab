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
external_url 'http://ip:8080' #外面port
nginx['listen_port'] = 80 #內部port
```

# 3. 備份gitlab

若要備份整個gitlab，先進去container內部，記得備份以下路徑內容：

```shell
/etc/gitlab/gitlab.rb 配置文件須備份 
/var/opt/gitlab/nginx/conf nginx配置文件 
/etc/postfix/main.cfpostfix 郵件配置備份
```

然後打上下面語法進行備份，預設備份路徑會在/var/opt/gitlab/backups：

```shell
gitlab-rake gitlab:backup:create
```
# 4. 還原gitlab

還原前需要先停止服務：

```shell
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
```

因為先前備份檔可能會有權限問題，先將權限改為777，前面的1481598919是備份編號，每個gitlab版本可能會有些差異。

```shell
chmod 777 /var/opt/gitlab/backups/1585927685_2020_04_03_11.1.4_gitlab_backup.tar
```
接下來用下面語法還原備份，一般來說會跳出確認選項，都打yes即可。
```shell
gitlab-rake gitlab:backup:restore BACKUP=1585927685_2020_04_03_11.1.4
```
還原完別忘了確認功能是否正常，包含前面說明的LDAP帳號連線狀況。

# 5. 升級gitlab

Gitlab升級是一件麻煩的事情，首先必須升級到該大版本的最後一個版本號，才能再升級大版本，詳細可以參考[官方文件](https://docs.gitlab.com/ee/policy/maintenance.html#upgrade-recommendations)。

升級前不要忘記備份，建議先建立一個新的container複製一份現有gitlab，升級完後再取代現有gitlab，即取代port號即可，避免升級失敗等問題。

首先先根據原本環境設定建一個純淨的gitlab container(id:3dd99e102e35)，然後把原本container(id:cb0ac58d01e0)的備份檔丟到外部VM，並且丟進去新的container進行還原，還原後一樣確認功能是否正常。

```shell
docker cp cb0ac58d01e0:/var/opt/gitlab/backups/1585927685_2020_04_03_11.1.4_gitlab_backup.tar ~/bk
docker cp ~/bk/1585927685_2020_04_03_11.1.4_gitlab_backup.tar 3dd99e102e35:/var/opt/gitlab/backups 
```


















