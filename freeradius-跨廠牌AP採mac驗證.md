# debian10 (netinstall) + freeradius3 建立MAC純文字認證  
### 資料日期2021-12


------------
### 以下指令使用root身分執行，需要sudo自行替換
------------

## 先更新  
apt update -y;apt dist-upgrade

## 安裝基本所需套件  
apt install net-tools freeradius -y

## 修改/etc/freeradius/3.0/clients.conf
-------------------------------------------------------
### 假設驗證密碼為radius8787  
### 假設做認證的設備(AP或者controller)網段來自192.168.0.0/16 

```
# IPv6 Client
client localhost_ipv6 {
        ipv6addr        = ::1
        secret          = radius8787
}
client private-network-1 {
        ipaddr          = 192.168.0.0/16
        secret          = radius8787        
}
```
##  修改/etc/freeradius/3.0/sites-available/default  
### (這邊設定如果SSID是來自TEST-DD的才允許驗證，增加安全性)  

```
server default {
listen {

        type = auth
        ipaddr = *
        port = 0

        limit {
              max_connections = 16
              lifetime = 0
              idle_timeout = 30
        }
}
listen {
        ipaddr = *
        port = 0
        type = acct
        limit {
        }
}
listen {
        type = auth
        ipv6addr = ::   # any.  ::1 == localhost
        port = 0
        limit {
              max_connections = 16
              lifetime = 0
              idle_timeout = 30
        }
}
listen {
        ipv6addr = ::
        port = 0
        type = acct
        limit {
        }
}

authorize {
preprocess
rewrite_calling_station_id
if ((&Calling-Station-Id) && ( &Called-Station-Id =~ /TEST-DD$/ )) {
                # Now check against the authorized_macs file
                authorized_macs

                if (!ok) {
                        reject
                }
                else {
                        # accept
                        update control {
                                Auth-Type := Accept
                        }
                }
        }



}

}
```



## 修改/etc/freeradius/3.0/mods-enabled/files  

```
files {
        moddir = ${modconfdir}/${.:instance}
        filename = ${moddir}/authorize
        acctusersfile = ${moddir}/accounting
        preproxy_usersfile = ${moddir}/pre-proxy
}



files authorized_macs {
        key = "%{Calling-Station-ID}"

        usersfile = ${confdir}/authorized_macs
}

```

## 新增/etc/freeradius/3.0/policy.d/mac-address.conf  
### 採用toupper統一把MAC改成大寫  
```
rewrite_calling_station_id {
        if (Calling-Station-Id =~ /([0-9a-f]{2})[-:]?([0-9a-f]{2})[-:.]?([0-9a-f]{2})[-:]?([0-9a-f]{2})[-:.]?([0-9a-f]{2})[-:]?([0-9a-f]{2})/i) {
                update request {
                        Calling-Station-Id := "%{toupper:%{1}-%{2}-%{3}-%{4}-%{5}-%{6}}"
                }
        }
        else {
                noop
        }
}

```




## 新增/etc/freeradius/3.0/authorized_macs作為MAC登記檔案  
### (假設裝置取名為iPad-06，MAC地址為90-B6-5F-46-DE-44(根據上方設定全用

```
#iPad-06
90-B6-5F-46-DE-44
```
------------
## !!!!!每次增加設備mac位址後記得要重啟服務!!!!!!!  

systemctl restart freeradius.service


------------




### 進入測試模式debug  
systemctl stop freeradius.service ; freeradius -X

### 確認一切正常後，啟動並常駐服務  
systemctl enable  freeradius.service



### fortigate對應方法
1.新增一個radius server  
User/Authentication新增一個radius server  
NAS IP 與 Primary Server IP都設定為freeradius3的IP  
Secret 就是上面的驗證碼radius8787  


2.新增一個使用mac address過濾的SSID  
SSID取名為TEST-DD (有檢查SSID來自TEST-DD的才進行驗證)  
模式選擇bridge，security選擇open  
Client MAC Address Filtering打勾 選取剛剛建立的radius server  
Optional VLAN ID自己決定，預設0等於走default vlan  


### unifi對應方法  
1.新增一個radius server  
從Profile新增一組radius  
IP Address就是freeradius的IP  
port 1812  
Secret就是上面的驗證碼radius8787  

2.新增一個使用mac address過濾的SSID  
Wireless Network Create 一個新的SSID，取為TEST-DD   (有檢查SSID來自TEST-DD的才進行驗證)  
ADVANCED OPTIONS裡面RADIUS MAC AUTHENTICATION 要enable  
Radius profile選剛剛建立那一個  
MAC Address Format選全大寫的 AA-BB-CC-DD-EE-FF  
vlan的部分從最上面的Network去選(要在Networks那邊有建立過才行)  

------------------


### 補充說明  
---------------------
如果要做到dynamic vlan ，根據mac address指派vlan   
需要wifi controller設定配合，目前測試unifi與foritgate皆可使用  
記得將ssid設定成為dynamic vlan   

freeradius則是要去修改authorized_macs檔案  
以下的例子是把iPad-06指定到vlan 60  

```
#iPad-06
90-B6-5F-46-DE-44
        Tunnel-Type = 13,
        Tunnel-Medium-Type = 6,
        Tunnel-Private-Group-ID = 60
```

