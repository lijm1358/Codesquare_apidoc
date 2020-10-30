# Codesquare_apidoc
keystone 유저 생성부터 vm생성까지 필요한 api 요청입니다.    
최종적으로 모든 요청을 다 보내게 되면 http://34.64.118.138:8989/[사용자id] 주소를 통해 ide를 띄울 수 있게 됩니다. 
## changelog
#### 10.29 추가
1. 모든 요청 url주소가 바뀌었습니다.(3.236.100.160 -> 34.64.118.138)
2. 계정 생성 시 default 프로젝트, 연결 프로젝트 및 일반 사용자 토큰 요청시 프로젝트명이 바뀌었습니다.(demo -> codesquare)
3. 인스턴스 생성 시 flavor 및 "availability_zone" 항목이 추가되었습니다.
4. 네트워크 관련 요청 url주소 포트가 바뀌었습니다.(9696 -> 9999)
#### 10.30
1. url ip주소에서 도메인으로 변경(34.64.118.138 -> stack.codesquare.space)
## 계정 생성

### admim 토큰 얻기

계정 생성 및 vm 생성 시 헤더에 사용자에 대한 토큰이 필요합니다.
계정 생성에는 admin 계정의 토큰이 필요하고, vm 생성시에는 vm을 생성하는 유저의 토큰이 필요합니다.
```bash
curl -i -H "Content-Type: application/json" \
-d '
{ "auth": {
    "identity": {
        "methods": ["password"],
            "password": {
                "user": {
                    "name": "admin",
                    "domain": { 
                        "id": "default" 
                       },
                    "password": "password"
                   }
              }
         },
        "scope": {
            "project": {
                "name": "admin",
                    "domain": { "id": "default" }
              }
         }
    }
}' \
"http://stack.codesquare.space/identity/v3/auth/tokens" | grep X-Subject-Token
```
위의 코드 실행 시 X-Subject-Token: [token] 형식으로 토큰 발급이 가능합니다.    
해당 토큰을 따로 변수 등($OS_TOKEN)으로 저장시켜서 사용하시면 될 거 같습니다.

### default project가 codesquare인 유저 생성
keystone 서버에 유저 생성 시 기본으로 연결할 project id가 필요합니다.
```bash
curl -X POST \
 -H "X-Auth-Token: $OS_TOKEN" \
 -H "Content-Type: application/json" \
 -d '
{"user": {
    "default_project_id": [project_id],
    "name": "newuser1234",
    "password": "qwerty"
    }
}' \
"http://stack.codesquare.space/identity/v3/users" | python -m json.tool
```

default_project_id에는 codesquare 프로젝트의 id값이 필요하므로, [project_id]는 아래와 같은 방식으로 얻을 수 있습니다.
```bash
curl -X GET \
 -H "X-Auth-Token: $OS_TOKEN" \
 "http://stack.codesquare.space/identity/v3/projects" | python -m json.tool | jq '.projects[]' | jq 'select(.name == "codesquare")' | jq '.id'
```

### 만든 유저를 codesquare 프로젝트에 할당 및 user 역할 배정
생성한 유저가 만든 vm은 codesquare 프로젝트에 저장되고, 유저에게는 일반 사용자인 user 역할을 배정해줍니다.
```bash
curl -s -X PUT \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/identity/v3/projects/[project_id]/users/[user_id]/roles/[role_id]"
```
[project_id]는 위에 언급한 방식으로,    
```bash
curl -X GET \
 -H "X-Auth-Token: $OS_TOKEN" \
 "http://stack.codesquare.space/identity/v3/projects" | python -m json.tool | jq '.projects[]' | jq 'select(.name == "codesquare")' | jq '.id'
```
[user_id]는
```bash
curl -X GET  -H "X-Auth-Token: $OS_TOKEN"  "http://stack.codesquare.space/identity/v3/users" | python -m json.tool | jq '.users[]' | jq 'select(.name == "[사용자id]")' | jq '.id'
```
[role_id]는
```bash
curl -X GET  -H "X-Auth-Token: $OS_TOKEN"  "http://stack.codesquare.space/identity/v3/roles" | python -m json.tool | jq '.roles[]' | jq 'select(.name == "user")' | jq '.id'
```
와 같은 방법으로 id값을 얻어올 수 있습니다.

## VM 생성

### 새롭게 만든 유저의 id, 비밀번호를 이용해 새 token 발급
vm 생성은 vm을 생성하려는 유저의 토큰을 이용해 생성되므로, 새롭게 생성한 유저의 토큰을 발급받습니다.
```bash
curl -i -H "Content-Type: application/json" \
-d '
{ "auth": {
    "identity": {
        "methods": ["password"],
            "password": {
                "user": {
                    "name": "newuser1234",
                    "domain": { 
                        "id": "default" 
                       },
                    "password": "qwerty"
                   }
              }
         },
        "scope": {
            "project": {
                "name": "codesquare",
                    "domain": { "id": "default" }
              }
         }
    }
}' \
"http://stack.codesquare.space/identity/v3/auth/tokens" | grep X-Subject-Token 
```

### 만든 유저의 token을 이용해 인스턴스 생성
인스턴스 생성을 위해 image id, flavor(사양) id, private network id, security group 이름이 필요합니다.
```bash
curl -g -i -X POST \
-H "User-Agent: python-novaclient" \
-H "Content-Type:application/json" \
-H "Accept: application/json" \
-H "X-Auth-Token:$OS_TOKEN" \
-d '
{"server": {
	"name":"newuser1234",
	"imageRef":[image_id],
	"flavorRef":"d2",
	"networks":[{"uuid":[network_id]}],
	"security_groups": [{"name": "cdr-rule"}],
	"availability_zone": "nova:codesquare-devstack-compute2"
	}
}' "http://stack.codesquare.space/compute/v2.1/servers"
```
name은 사용자id로 해주시면 됩니다.    
imageRef는 code-server가 설치되어있는 ubuntucdr-1.0이라는 이름의 이미지를 사용할것이며, 다음과 같은 요청으로 [image_id]를 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/images" | python -m json.tool | jq '.images[]' | jq 'select(.name == "ubuntucdr-1.0")' | jq '.id'
```
flavorRef는 ds1G (vCPU:1, RAM:1 GB, HDD: 10 GB) 사양을 이용할것이며 위처럼 d2로 설정해주면 됩니다.
만약, 사양 변경 시 다음과 같은 요청으로 flavor 목록을 확인할 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/flavors" | python -m json.tool
```
networks의 uuid는 codesquare 프로젝트에 설정되어있는 heat-net 네트워크를 사용할 것이며, 다음과 같은 요청으로 [network_id]를 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space:9999/v2.0/networks" | python -m json.tool | jq '.networks[]' | jq 'select(.name == "heat-net")' | jq '.id'
```
* 인스턴스 생성 시, 유동 ip 할당을 위해 **인스턴스 id**를 미리 저장해둡니다.
## VM 설정 및 외부 ip 주소와 코드서버 연결

### 생성된 인스턴스에 유동 ip 할당
외부에서 openstack 인스턴스로 접근하기 위해 유동 ip를 할당해줍니다.
```bash
curl -s \
 -H "X-Auth-Token: $OS_TOKEN" \
 -H "Content-Type: application/json" \
 -d '
{"floatingip": {
    "floating_network_id":[network_id],
    "fixed_ip_address":[address],
    "port_id":[port_id]
    }
}' "http://stack.codesquare.space:9999/v2.0/floatingips" | python -m json.tool
```
floating_network_id는 유동 ip 주소를 받을 네트워크(public 네트워크)의 id이며, 다음과 같은 요청으로 [network_id]를 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space:9999/v2.0/networks" | python -m json.tool | jq '.networks[]' | jq 'select(.name == "public")' | jq '.id'
```
fixed_ip_address는 인스턴스 생성 시 자동으로 할당되는 내부 ip주소이며, 다음과 같은 요청으로 [address]를 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/servers/[instance id]" | python -m json.tool | jq '.server.addresses' | jq '.["heat-net"][]' | jq '.addr'
```
* [instance id]는 생성한 instance의 id값을 넣어주면 됩니다. 
* (10.28 추가) 네트워크 구조 변경에 따라 내부 ip주소를 찾는 방법이 약간 달라졌습니다. 위의 url로 GET 요청 시 아래와 같은 response를 얻을 수 있습니다.
```JSON
{
    "server": {
        "OS-DCF:diskConfig": "MANUAL",
        "OS-EXT-AZ:availability_zone": "nova",
        "OS-EXT-STS:power_state": 1,
        "OS-EXT-STS:task_state": null,
        "OS-EXT-STS:vm_state": "active",
        "OS-SRV-USG:launched_at": "2020-10-28T13:28:42.000000",
        "OS-SRV-USG:terminated_at": null,
        "accessIPv4": "",
        "accessIPv6": "",
        "addresses": {
            "heat-net": [
                {
                    "OS-EXT-IPS-MAC:mac_addr": "fa:16:3e:00:f6:22",
                    "OS-EXT-IPS:type": "fixed",
                    "addr": "10.0.5.181",
                    "version": 4
                },
                {
                    "OS-EXT-IPS-MAC:mac_addr": "fa:16:3e:00:f6:22",
                    "OS-EXT-IPS:type": "floating",
                    "addr": "192.168.1.184",
                    "version": 4
                }
            ]
        },
```
    
port_id는 생성한 instance 인터페이스의 포트 id이며, 다음과 같은 요청으로 [port_id]를 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space:9999/v2.0/ports" | python -m json.tool | jq '.ports[]' | jq 'select(.fixed_ips[].ip_address == [fixed_addr])' | jq '.id'
```
[fixed_addr]는 위의 fixed_ip_address에 들어가는 [address]값(인스턴스의 내부 고정 ip주소)을 넣어주면 됩니다.

### 할당된 유동 ip와 유저 id로 code-server에 접속할 수 있는 링크 생성
```bash
curl -X POST \
-H "Content-Type: application/json" \
-d '
{"name": "newuser1234", "addr": [floating_addr]}
' "http://34.64.118.138:8890/urlinfo"
```
name에는 사용자id를, [floating_addr]은 위에서 할당된 floating ip 값을 넣어주면 됩니다.

위의 과정을 거치고 나면 http://ide.codesquare.space:8989/newuser1234/ 으로 ide에 접속할 수 있게 됩니다.

## VM 삭제

### 유동 ip 해제 및 삭제
```bash
curl -X DELETE \
-H "X-Auth-Token: $OS_TOKEN" \
-H "Accept: application/json" \
"http://stack.codesquare.space:9999/v2.0/floatingips/[floatingIp_id]" | python -m json.tool
```
[floatingIp_id]는 다음과 같은 방법으로 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space:9999/v2.0/floatingips" | python -m json.tool | jq '.floatingips[]' | jq 'select(.floating_ip_address == [floating_ip])' | jq '.id'
```
[floating_ip]는 다음과 같은 방법으로 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/servers/7fc53fb9-2fbc-43ae-9759-0c9c85c5933c" | python -m json.tool | jq '.server.addresses' | jq '.["heat-net"][]' | jq 'select(."OS-EXT-IPS:type" == "floating")' | jq '.addr'
```

### Openstack Instance 삭제
```bash
curl -X DELETE \
-H "X-Auth-Token:$OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/servers/[Instance_Id]"
```
[Instance_Id]는 다음과 같은 방법으로 얻을 수 있습니다.
```bash
curl -X GET \
-H "X-Auth-Token: $OS_TOKEN" \
"http://stack.codesquare.space/compute/v2.1/servers?all_tenants" | python -m json.tool | jq '.servers[]' | jq 'select(.name == [사용자id])' | jq '.id'
```

### url 연결 해제
```bash
curl -X DELETE \
"http://stack.codesquare.space:8890/urlinfo/[사용자id]"
```

