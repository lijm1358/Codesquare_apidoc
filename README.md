# Codesquare_apidoc
keystone 유저 생성부터 vm생성까지 필요한 api 요청입니다.    
최종적으로 모든 요청을 다 보내게 되면 http://3.235.236.245:8989/[사용자id] 주소를 통해 ide를 띄울 수 있게 됩니다.

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
                    "password": "1234"
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
"http://3.235.236.245/identity/v3/auth/tokens" | grep X-Subject-Token
```
위의 코드 실행 시 X-Subject-Token: [token] 형식으로 토큰 발급이 가능합니다.    
해당 토큰을 따로 변수 등($OS_TOKEN)으로 저장시켜서 사용하시면 될 거 같습니다.

### default project가 demo인 유저 생성
keystone 서버에 유저 생성 시 기본으로 연결할 project id가 필요합니다.
```bash
curl -X POST \
 -H "X-Auth-Token: $OS_TOKEN" \
 -H "Content-Type: application/json" \
 -d '
{"user": {
    "default_project_id": [project id],
    "name": "newuser123",
    "password": "qwerty"
    }
}' \
"http://3.235.236.245/identity/v3/users" | python -m json.tool
```

default_project_id에 들어갈 값은 아래와 같은 방식으로 얻을 수 있습니다.
```bash
curl -X GET \
 -H "X-Auth-Token: $OS_TOKEN" \
 "http://3.235.236.245/identity/v3/projects" | python -m json.tool | jq '.projects'
```

만든 유저를 demo 프로젝트와 user 역할 배정
curl -s -X PUT \
-H "X-Auth-Token: $OS_TOKEN" \
"http://3.235.236.245/identity/v3/projects/d621b4862c984d248a525641ec7d1fdd/users/90a8c2c3dbe74a3c96ce234517bcde14/roles/247eee7a7bd74239a8351aa6c7c97f03"

새롭게 만든 유저의 id, 비밀번호를 이용해 새 token 발급
curl -i -H "Content-Type: application/json" \
-d '
{ "auth": {
    "identity": {
        "methods": ["password"],
            "password": {
                "user": {
                    "name": "newuser123",
                    "domain": { 
                        "id": "default" 
                       },
                    "password": "qwerty"
                   }
              }
         },
        "scope": {
            "project": {
                "name": "demo",
                    "domain": { "id": "default" }
              }
         }
    }
}' \
"http://3.235.236.245/identity/v3/auth/tokens" | grep X-Subject-Token 

만든 유저의 token을 이용해 인스턴스 생성
curl -g -i -X POST \
-H "User-Agent: python-novaclient" \
-H "Content-Type:application/json" \
-H "Accept: application/json" \
-H "X-Auth-Token:$OS_TOKEN" \
-d '
{"server": {
	"name":"user123",
	"imageRef":"d64b6037-2952-4644-9cd7-b6e873c47b34",
	"flavorRef":"d3",
	"networks":[{"uuid":"a2358c05-9dc8-401a-9543-5d587ace3e19"}],
	"security_groups": [{"name": "default"}]
	}
}' "http://3.235.236.245/compute/v2.1/servers"

생성된 인스턴스에 유동 ip 할당
curl -s \
 -H "X-Auth-Token: $OS_TOKEN" \
 -H "Content-Type: application/json" \
 -d '
{"floatingip": {
    "floating_network_id":"cacb76d0-dad9-41cf-85c9-665ca302b490",
    "fixed_ip_address":"10.0.0.17",
    "port_id":"2b2c9bf7-bbf0-4b7d-826f-6b1a24c9ae2f"
    }
}' "http://3.235.236.245:9696/v2.0/floatingips" | python -m json.tool

할당된 유동 ip와 유저 id로 code-server에 접속할 수 있는 링크 생성
curl -X POST \
-H "Content-Type: application/json" \
-d '
{"name": "user123", "addr": "172.24.4.12"}
' "http://3.235.236.245:8890/urlinfo"

그럼 이제 3.235.236.245:8989/user123/으로 ide를 띄울 수 있습니다.
