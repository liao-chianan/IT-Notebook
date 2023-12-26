# debian10 (netinstall)+ freeradius3 + 802.1X 搭配Windows/Samba AD進行MSCHAPv2驗證  
### 資料日期2021-12

------------
## 環境設定參數  
### SAMBA AD： netbios name 為smbdc 、IP為192.168.88.3  
### 本台RADIUS： netbios name 為radius2 、IP為192.168.88.4  
### 測試帳號與密碼：aaa / aaa  
### 網域為：yyps.tp.edu.tw  
------------
### 以下指令使用root身分執行，需要sudo自行替換
------------

## 先更新  
apt update -y;apt dist-upgrade -y  
  
## 安裝基本所需套件  
apt install net-tools freeradius  curl lsb-release dnsutils -y

## 修改/etc/resolv.conf  將DNS設為SAMBA AD主機的IP 並補上網域資料   
nameserver 192.168.88.3  
search yyps.tp.edu.tw  
domain yyps.tp.edu.tw  

## 修改/etc/hosts 補上自己的domain以及SAMBA的domain   
-------------------------------------------
/etc/hosts/的內容如下
   
192.168.88.4  radius2.yyps.tp.edu.tw radius2  
192.168.88.3  smbdc smbdc.yyps.tp.edu.tw  


修改完可以ping  SAMBA AD的netbios name確認是否正確

## 安裝samba與winbind相關套件  
apt install samba-common winbind krb5-config libpam-winbind libnss-winbind resolvconf -y

### 安裝過程會提示輸入realm  
Default Kerberos version 5 realm 輸入yyps.tp.edu.tw

## 安裝完畢後修改/etc/krb5.conf  
-------------------------------------------------
檔案開頭加上  
[libdefaults]  
       dns_lookup_realm = false  
        dns_lookup_kdc = true  
        default_realm = YYPS.TP.EDU.TW  

[realms]補上  
 YYPS.TP.EDU.TW = {  
                kdc = smbdc.yyps.tp.edu.tw  
                admin_server = smbdc.yyps.tp.edu.tw  
        }  

[domain_realm]的區塊最後補上  
       .yyps.tp.edu.tw = YYPS.TP.EDU.TW  
        yyps.tp.edu.tw = YYPS.TP.EDU.TW  



## 修改/etc/samba/smb.conf  
-------------------------------------------
/etc/samba/smb.conf內容如下
[global]  
       security = ADS  
       workgroup = YYPS  
       realm = YYPS.TP.EDU.TW  
       client NTLMv2 auth = Yes  
       log file = /var/log/samba/%m.log  
       log level = 1  
       password server = smbdc.yyps.tp.edu.tw  
       winbind use default domain = true  
       winbind offline logon = false  
       template homedir = /home/%U  
       template shell = /bin/bash  
       idmap config * : backend = tdb  
       idmap config * : range = 10000-20000  


修改/etc/nsswitch.conf改變認證方式  
---------------------------------------
/etc/nsswitch.conf內容如下  

passwd:         compat winbind  
group:          compat winbind  
shadow:         compat winbind  
gshadow:        files  
hosts:          files dns  
networks:       files  
protocols:      db files  
services:       db files  
ethers:         db files  
rpc:            db files
netgroup:       nis  



## 加入AD網域(需要輸入密碼)  
net ads join -U Administrator  

## 重啟winbind  
systemctl restart winbind  

## 確認群組與使用者是否正確  
wbinfo -u  
wbinfo -g  

## 測試ntlm_auth帳密有沒有正確work  
ntlm_auth  --username=aaa --password=aaa  

## 補上權限設定  
### 讓freerad加入winbind群組，修正winbindd pipe的權限
### (不然會看到freeradius出現Reading winbind reply failed錯誤訊息)   

usermod -a -G winbindd_priv freerad  
chown root:winbindd_priv /var/lib/samba/winbindd_privileged/  

## 重啟winbind服務  
systemctl restart winbind  

## 進行freeradius的設定
記得要先到SAMBA AD主機修改smb.conf 補上
ntlm auth = Yes
client NTLMv2 auth = yes
這樣freeradius才能使用ntlm  以及 ntlmv2進行驗證





## 修改freeradius設定檔案
-----------------------------------------------------
每次修改完可以 /usr/sbin/freeradius -X     進入debug模式，驗證設定是否正確  



## 修改/etc/freeradius/3.0/clients.conf    
-------------------------------------------------------
### 假設驗證密碼為radius8787  
### 假設做認證的設備(AP或者controller)網段來自192.168.0.0/16 
``` 
client localhost_ipv6 {  
        ipv6addr        = ::1  
        secret          = radius8787  
}
client private-network-1 {  
        ipaddr          = 192.168.0.0/16  
        secret          = radius8787          
}  
```

## 修改/etc/freeradius/3.0/mods-available/mschap  
-----------------------------------------------------------
### 假設使用的winbind 網域為YYPS
```
mschap {
    use_mppe=yes
    require_encryption = yes
    require_strong = yes
    with_ntdomain_hack = yes

    winbind_username = "%{mschap:User-Name}"
    winbind_domain = "YYPS"

   ntlm_auth = "/usr/bin/ntlm_auth --allow-mschapv2 --request-nt-key --username=%{%{Stripped-User-Name}:-%{%{User-Name}:-None}} --challenge=%{%{mschap:Challenge}:-00} --nt-response=%{%{mschap:NT-Response}:-00}"

}
```

##修改 /etc/freeradius/3.0/mods-enabled/eap  
-----------------------------------------------------
把default_eap_type = md5 改成mschapv2



### 如果要限制特定群組(像是WIFI)才能驗證通過
可以在上面的ntlm_auth參數補上--require-membership-of='YYPS\WIFI'
這樣就會要求有加入WIFI群組的使用者才能驗證通過(同一個使用者可以加入多個群組)  


### 如果要針對特別使用者作限制或者發下vlan  
要去修改/etc/freeradius/3.0/sites-available/default的post auth部分
例如拒絕bbb這個使用者連線  
```
post-auth {
if (&User-Name  =~ /bbb$/) {
	reject
  }
```

### 例如指定bbb這個使用者使用vlan 100
```
post-auth {
if (&User-Name  =~ /bbb$/) {
	update reply { 
    &Tunnel-Type = 13, 
    &Tunnel-Medium-Type = 6,
    &Tunnel-Private-Group-Id = "100"
    }
  }
}
```

## 修改/etc/freeradius/3.0/mods-available/ntlm_auth  
把mydomain改成YYPS  



## 進入測試模式debug  
systemctl stop freeradius.service ; freeradius -X

## 由radius本機測試mschap是否正常動作  
(localhost的預設secret 為testing123)  
```
radtest -t mschap aaa "aaa" localhost:18120 0 testing123
```
(成功會出現Received Access-Accept Id )


## 確認一切正常後，啟動並常駐服務  
systemctl start  freeradius.service  
systemctl enable  freeradius.service  

## 如果要log認證紀錄，修改/etc/freeradius/3.0/radiusd.conf  
-----------------------------------------------------
```
log {
    destination = files
    file = ${logdir}/radius.log	
    syslog_facility = daemon
    stripped_names = yes
    auth = yes
    auth_badpass = yes
    auth_goodpass = yes
#   msg_goodpass = ""
#   msg_badpass = ""
}
```

## 如果要詳細的log紀錄，修改/etc/raddb/sites-enabled/default  
------------------------------------------------------
在authorize{}區塊的最後面補上auth_log
在post-auth{}區塊的最後面補上reply_log



---------------------------------------------------------------
------------------------------------------------------------


### [fortigate對接freeradius方法]  
1.新增一個radius server  
User/Authentication新增一個radius server  
NAS IP 與 Primary Server IP都設定為freeradius的IP  
Authentication method選MS-CHAP-v2  
Secret 就是上面的驗證碼radius8787  


2.新增一個使用802.1X搭配AD過濾的SSID  
SSID取名為TEST-8021xAD   
模式選擇bridge，security選擇WPA2 Enterprise  
Client MAC Address Filtering打勾 選取剛剛建立的radius server  
Optional VLAN ID自己決定，預設0等於走default vlan  


### [unifi對接freeradius方法]  
1.新增一個radius server  
從Profile新增一組radius  
IP Address就是freeradius的IP  port 1812  
Secret就是上面的驗證碼radius8787  

2.新增一個使用mac address過濾的SSID  
Wireless Network Create 一個新的SSID，取為TEST-8021xAD    
vlan的部分從最上面的Network去選(要在Networks那邊有建立過才行)  
 


### [IOS 連線]
打開WIFI設定直接連TEST-8021xAD，輸入AD的帳密，信任憑證後就可連線


### [Android連線]
打開WIFI設定直接連TEST-8021xAD
EAP方法選PEAP
階段2驗證選MSCHAPV2
CA憑證選不進行驗證
身分輸入AD帳號
匿名身分留空
密碼輸入AD密碼



## 如果希望取得AD使用者群組分開處理vlan，須加上ldap模組與設定   

先安裝ldap模組  
apt install -y freeradius-ldap  
修改/etc/freeradius/3.0/mods-available/ldap     
補上下方修正ADS SERVER參數  


在ldap{}區塊修改參數(假設AD的ip為192.168.88.3,administrator密碼為12345678)

```
    server = '192.168.88.3'
    identity = 'cn=administrator,cn=users,dc=yyps,dc=tp,dc=edu,dc=tw'
    password = 12345678
    base_dn = 'dc=yyps,dc=tp,dc=edu,dc=tw'
    port = 389
	
```

#### 找到 edir = no  這邊記得uncomment  
--------------------------------------------

底下的設定記得uncomment
```
update {}

       control:Password-With-Header    += 'userPassword'
       control:NT-Password     := 'ntPassword'
       reply:Reply-Message     := 'radiusReplyMessage'
       reply:Tunnel-Type       := 'radiusTunnelType'
       reply:Tunnel-Medium-Type    := 'radiusTunnelMediumType'
       reply:Tunnel-Private-Group-ID   := 'radiusTunnelPrivategroupId'
	   reply:memberOf                  += 'memberOf'
	   reply:displayName               += 'displayName'

        #  Where only a list is specified as the RADIUS attribute,
        #  the value of the LDAP attribute is parsed as a valuepair
        #  in the same format as the 'valuepair_attribute' (above).
        control:            += 'radiusControlAttribute'
        request:            += 'radiusRequestAttribute'
        reply:              += 'radiusReplyAttribute'
```   

#### 在user區塊確認有修改成sAMAccountName
```
user{} 裡面

base_dn = "${..base_dn}"
filter = "(sAMAccountName=%{%{Stripped-User-Name}:-%{User-Name}})" 
```


####  在 options 區塊修改chase_referrals避免搜尋DN失敗 
```
options {}
 
chase_referrals = no  
rebind = yes  

```


### 修改完後建立連結啟用模組  
ln -s /etc/freeradius/3.0/mods-available/ldap   /etc/freeradius/3.0/mods-enabled/ldap  

### 如果有做reply:memberOf和reply:displayName  記得要補上attritube   
### 修改/etc/freeradius/3.0/dictionary   
```

ATTRIBUTE       memberOf        3001    string  
ATTRIBUTE       displayName        3002    string  
```

### 再回去修改/etc/freeradius/3.0/sites-available/default  
在post-auth 區塊補上判斷式，以底下的例子來說，屬於WIFI群組的人就會回應OKOK訊息，然後給予vlan100

------------------------------------
```
if (LDAP-Group == "WIFI") {

    update reply {

        &reply: += &session-state:
        Reply-Message += "okok"
		Reply-Message += "%{reply:memberOf}"
		Reply-Message += "%{reply:displayName}"
        &Tunnel-Type = 13,
        &Tunnel-Medium-Type = 6,
        &Tunnel-Private-Group-Id = "100"
    }
```


------------------------------------
```
在authorize {}區塊最後面補上判斷式，才有辦法回傳ldap撈到的資料
        ldap
           if ((ok || updated) && User-Password) {
              update {
              control:Auth-Type := ldap
             }
        }

```


## 如果要呼叫自訂的外部script 去修改reply message  

(假設呼叫/etc/freeradius/3.0/test.sh 內容 (注意目錄和權限))

echo "haha"
exit 0

#### exec模組要先開啟wait  
/etc/freeradius/3.0/mods-available/exec裡面的  
wait = yes  

&reply:Reply-Message += "%{exec:/bin/bash   /etc/freeradius/3.0/test.sh}"  

#### 這樣Reply-Message就會多一行haha (注意exec這個mods不支援中文)





## 2021因應Android 11 QPR1的安全性更新，已無法繼續使用[不進行驗證]

方法是建立DNS後跟letsencrypt申請免費憑證，然後補到freeradius裡面
參考文件https://framebyframewifi.net/2017/01/29/use-lets-encrypt-certificates-with-freeradius/

先拿到憑證(本基直接索取或者跟其他台已經有憑證的機器拷貝皆可)
假設憑證對應的域名是aaa.yyps.tp.edu.tw
把憑證檔案privkey.pem還有fullchain.pem
複製到/etc/freeradius/3.0/cert/letsencrypt中

修改 /etc/freeradius/3.0/mods-enabled/eap  檔案
找到tls-config tls-common 區塊

private_key_file = /etc/ssl/private/ssl-cert-snakeoil.key 
改成跟letsencrypt拿到的/etc/freeradius/3.0/certs/letsencrypt/privkey.pem

certificate_file = /etc/ssl/certs/ssl-cert-snakeoil.pem   
改成跟letsencrypt拿到的/etc/freeradius/3.0/certs/letsencrypt/fullchain.pem

重啟radius服務
systemctl restart freeradius

[Android11的新連線方式]
打開WIFI設定直接連TEST-8021xAD
EAP方法選[PEAP]
階段2驗證選[MSCHAPV2]
CA憑證選[使用系統憑證]
線上憑證狀態選[要求取得憑證狀態]
網域輸入aaa.yyps.tp.edu.tw
身分輸入AD帳號
匿名身分留空
密碼輸入AD密碼


-------------------------------------------------
## 利用eapol_test工具測試 peap+mschap完整流程  

安裝eapoltest需要debian的unstable repo

修改/etc/apt/sources.list加上兩行
  
deb https://deb.debian.org/debian unstable main contrib non-free  
deb-src https://deb.debian.org/debian unstable main contrib non-free  

### 接著安裝eapoltest  
apt install eapoltest

### eapoltest需要自己建立設定檔案eapol-yyps.txt，範例如下  
(假設帳號aaa 密碼aaa  使用的ssid為example)

```
network={
        ssid="example"
        key_mgmt=WPA-EAP
        eap=PEAP
        identity="aaa"
        anonymous_identity="anonymous"
        password="aaa"
        phase2="auth=MSCHAPV2"
}
```

測試的指令如下，如果過程都沒問題，最後會跳出success  
eapol_test -a 192.168.88.4 -c eapol-yyps.txt  -s radius8787  


## 如果要測試server端的TLS加有效憑證，可以先裝firefox取得DST_Root_CA_X3的憑證用來測試
apt install firefox  

然後設定檔案eapol-yyps.txt改成
```
network={
        ssid="example"
        key_mgmt=WPA-EAP
        eap=TTLS
        identity="aaa"
        anonymous_identity="anonymous"
        password="aaa"
        phase2="auth=MSCHAPV2"
		ca_cert="/usr/share/ca-certificates/mozilla/DST_Root_CA_X3.crt"
}
```

測試指令同上  
eapol_test -a 192.168.88.4 -c eapol-yyps.txt  -s radius8787  




