## 自建Freeradius 後端採用Google Workspace帳號，進行wifi 802.1x 認證   

### --資料日期2025.01
### --以下指令使用root身分執行，需要sudo自行替換
### --系統環境debian12.9 (pve8.3 + lxc)，Google Workspace教育版，測試環境android 15 / ios 18 / windows10/11

## 參考文件：  
https://support.google.com/a/answer/9048541?hl=zh-Hant  

https://nasirhafeez.splashnetworks.co/freeradius-with-google-g-suite-workspace-secure-ldap-for-wpa2-enterprise-wifi/  

https://www.alextwl.idv.tw/memo/2016/02/freeradius-eap-gtc-or-eap-ttls-with-inner-pap/  


## --先更新--  
apt update -y;apt dist-upgrade -y

##--調整時區以及語系--  
dpkg-reconfigure tzdata  
dpkg-reconfigure locales  

## --安裝基本所需套件--  
apt install curl wget sudo  net-tools freeradius freeradius-ldap freeradius-utils ldap-utils libldap-common  eapoltest -y

## --取得google ldap 憑證--   
 (會得到crt憑證和ke金鑰兩個檔案，有效期3年，以及開設的一組驗證帳號密碼)  
參考
https://support.google.com/a/answer/9048541?hl=zh-Hant





##確認ssl連接是否正常(正確訊息Verify return code: 0 (ok))
openssl s_client -connect ldap.google.com:636

##把取得的憑證傳到你的radius上  

假設憑證改名後放在/etc/freeradius/3.0/certs/ldap-client.crt  
然後金鑰改名後放在/etc/freeradius/3.0/certs/ldap-client.key

## --用指令測試憑證ldap憑證金鑰與帳號密碼是否能使用--  
(成功會列出網域內使用者)

LDAPTLS_CERT=/etc/freeradius/3.0/certs/ldap-client.crt LDAPTLS_KEY=/etc/freeradius/3.0/certs/ldap-client.key ldapsearch -x  -H ldaps://ldap.google.com:636 -b ou=users,dc=yyps,dc=tp,dc=edu,dc=tw -D '從google取得的驗證帳號' -w '從google取得的驗證密碼'


## --開始進行freerasiu設定檔修改--  

#### 修改允許存取radius的ip來源與認證碼(你的AP使用的IP範圍)  

nano /etc/freeradius/3.0/clients.conf  

---------------------------------------------
直接找地方加上  
```
client yyps-ap {
       ipaddr          = 192.168.0.0/16
       secret          = testing123
}
```
---------------------------------------------

#### 修改主要認證設定default檔案  

nano /etc/freeradius/3.0/sites-enabled/default  
在authroize區塊的pap(471行左右)下方加上  

------------------------------------------------------
```
          if (!control:Auth-Type && User-Password) {
            update control {
                   Auth-Type := ldap
            }
        }
```
-------------------------------------------------------

在authenticate區塊的Auth-Type PAP(537行左右)內容改為ldap  

------------------------------------------------------ 
```
        Auth-Type PAP {
                ldap
        }
```
------------------------------------------------------

在authenticate區塊Auth-Type LDAP (585行左右)把註解符號 # 都刪除  

------------------------------------------------------
```
        Auth-Type LDAP {
                ldap
        }
```
------------------------------------------------------


#### 複製ldap模組設定檔案並修改內容(記得改存取權限給freerad)  
cp /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/ldap  
chown freerad:freerad  
/etc/freeradius/3.0/mods-enabled/ldap
 

nano /etc/freeradius/3.0/mods-enabled/ldap 
修改以下參數  

--------------------------------------------------------  
(19行)  
server = 'ldaps://ldap.google.com:636'

(28行左右把註解符號 # 都刪除)  
identity = '從google取得的驗證帳號'  
password = '從google取得的驗證密碼'  

(33行)  
base_dn = 'ou=users,dc=yyps,dc=tp,dc=edu,dc=tw'

tls區塊的認證(603行左右把註解符號 # 刪除後修改)  
certificate_file = /etc/freeradius/3.0/certs/ldap-client.crt  
private_key_file = /etc/freeradius/3.0/certs/ldap-client.key  

接著在(618行左右把註解符號 # 刪除後修改)  

require_cert    = 'allow'  

--------------------------------------------------------  


#### 修改eap模組參數  

nano /etc/freeradius/3.0/mods-enabled/eap  

---------------------------------------------------------
eap區塊中把預設的eap模式改為peap
(27行)  
default_eap_type = peap

gtc區塊把 auth_type改為 ldap
(145行)  
auth_type = ldap


走inner-tunnel所以將peap區塊預設的eap改成gtc
(958行)  
default_eap_type = gtc

---------------------------------------------------------


#### 修改inner-tunnel設定(開放ldap)  

nano /etc/freeradius/3.0/sites-enabled/inner-tunnel   

-------------------------------------------------------------

(241行把這邊取消註解)  
```
        Auth-Type LDAP {
                ldap
        }
```
--------------------------------------------------------------

#### 修改proxy裡面的ream設定(使用者輸入完整帳號@域名會自動去掉@域名)  

(如果不設定，使用者只能輸入前面帳號，輸入完整帳號@域名會認證失敗)  

nano /etc/freeradius/3.0/proxy.conf

-------------------------------------------------------------------
在檔案最後面加上
```

realm yyps.tp.edu.tw {

}
```
------------------------------------------------------------------



####  如果要詳細的log認證紀錄，修改/etc/freeradius/3.0/radiusd.conf  
-----------------------------------------------------
(276行開始)
```
log {
    destination = files
    file = ${logdir}/radius.log	
    syslog_facility = daemon
    stripped_names = yes
    auth = yes
    auth_accept = yes
    auth_reject = yes
    auth_badpass = yes
    auth_goodpass = yes
#   msg_goodpass = ""
#   msg_badpass = ""
}
```

####  如果要詳細的log檔，修改/etc/freeradius/3.0/sites-enabled/default   

-------------------------------------------------------  

329行的 auth_log 取消註解  
853行的 reply_log 取消註解

-------------------------------------------------------  


**這樣未來在/var/log/freeradius/radacct/裡面會有更詳細log**  

.    
 

## --測試設定檔是否能成功運作--  
systemctl stop freeradius  
freeradius -XC 

重新啟動服務  
systemctl start freeradius  

或進入debug模式運作方便觀測  
freeradius -X  
  
.  
### 補充：針對不同google workspace群組允許放行與設定vlan  
**(此區段非必要，根據學校需求再設定)**  

#### 先補上ldap回傳參數屬性  
nano /etc/freeradius/3.0/dictionary  

-------------------------------------------------------------

最後面加上兩行  
```
ATTRIBUTE       memberOf        3001    string   
ATTRIBUTE       displayName     3002    string  
```
-------------------------------------------------------------  


#### 設定ldap回傳相關參數   
nano /etc/freeradius/3.0/mods-enabled/ldap  
  
-------------------------------------------------------------  
(128行左右取消註解並加上兩行)
```
update {
		control:Password-With-Header	+= 'userPassword'
		control:NT-Password		:= 'ntPassword'
		reply:Reply-Message		:= 'radiusReplyMessage'
		reply:Tunnel-Type		:= 'radiusTunnelType'
		reply:Tunnel-Medium-Type	:= 'radiusTunnelMediumType'
		reply:Tunnel-Private-Group-ID	:= 'radiusTunnelPrivategroupId'
                reply:memberOf                  += 'memberOf'
                reply:displayName               := 'displayName'
         

		#  Where only a list is specified as the RADIUS attribute,
		#  the value of the LDAP attribute is parsed as a valuepair
		#  in the same format as the 'valuepair_attribute' (above).
		control:			+= 'radiusControlAttribute'
		request:			+= 'radiusRequestAttribute'
		reply:				+= 'radiusReplyAttribute'
	}
```
-------------------------------------------------------------  

#### 從inner-tunnel 決定是否放行與設定vlan  
(假設teaching@yyps.tp.edu.tw群組放行，動態指定vlan id為90)  
(並且拒絕讓testgroup@yyps.tp.edu.tw群組認證)  


nano /etc/freeradius/3.0/sites-enabled/inner-tunnel  


------------------------------------------------------  
**(283行左右) post-auth區段裡面加上**
```

if (LDAP-Group == "teaching") {

    update outer.session-state {
        Reply-Message += "%{reply:memberOf[*]}"		
        Reply-Message += "%{reply:displayName}",		
        Reply-Message += "教師群組teaching",        
        &Tunnel-Type = 13,
        &Tunnel-Medium-Type = 6,
        &Tunnel-Private-Group-Id = "90"
		    }
   }
   
   
if (LDAP-Group == "testgroup") {
    reject    
   }
```

**(注意! 如果使用者同時屬於兩個群組，只要其中一個群組reject就會拒絕驗證)**  

-----------------------------------------------------  





### 2021之後，因應Android 11 QPR1的安全性更新，已無法繼續使用[不進行驗證]，
使用者連線時要選 **[首次使用時信任]** ，不然就需要用底下的方式取得有效憑證

方法是建立DNS後跟letsencrypt申請免費憑證，然後補到freeradius裡面，參考文件  

https://framebyframewifi.net/2017/01/29/use-lets-encrypt-certificates-with-freeradius/  


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


**[IOS的連線方式]**  
輸帳密然後信任憑證即可  

**[Android11以後的連線方式]**  

EAP方法選[PEAP]  
階段2驗證選[GTC]  

CA憑證選[使用系統憑證]或者[首次信任]  
線上憑證狀態選[要求取得憑證狀態]--(使用系統憑證才要)  
網域輸入aaa.yyps.tp.edu.tw--(使用系統憑證才要)  
身分輸入Google網域使用者帳號  
匿名身分留空  
密碼輸入Google網域使用者密碼  

**[Windows的連線方式]**  
windows作業系統沒有原生支援peap + gtc，因此需要安裝GTC client，安裝與設定可以參考這邊
https://wireless.kh.edu.tw/archives/250/


#### 從另一台linux機器測試的方式  
用radtest + pap驗證 以及 eapoltest + ttls + gtc驗證  
測試是否能運作正常  


**明碼測試指令**  
radtest '使用者google帳號' '使用者google密碼' 192.168.1.xx:1812 0 testing123  

**成功會回傳Received Access-Accept**  

**eapol加密測試**  
eaptol設定檔案yyps-google-test.conf內容  

----------------------------------  
```
network={
        ssid="example"
        key_mgmt=WPA-EAP
        eap=PEAP
        identity="使用者google帳號"
        anonymous_identity="使用者google帳號"
        password="使用者google密碼"
        phase2="auth=GTC"
}
```
----------------------------------  
**eapol 測試指令**  

eapol_test -a 192.168.1.xx -c yyps-google-test.conf  -s testing123 -N '31:s:88:88:88:99:99:99-Google-Radius'  

**成功會回傳SUCCESS**  

P.S. 只用fortigate的AP，要拉google ldap進來做驗證，不用特別架設radius，直接參考底下這篇就好

https://community.fortinet.com/t5/FortiGate/Technical-Tip-Google-Suite-LDAP-integration-with-FortiGate-using/ta-p/244848

