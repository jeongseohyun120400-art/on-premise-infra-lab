# Lab Infrastructure as Code

노트북 한 대(RAM 12GB, i5-1035G4)에서 VMware Workstation Pro로 4대의 VM을 구축하고,
Vagrant + Ansible로 전체 구성을 코드화(IaC)한 개인 인프라 랩입니다.
수동 구축으로 동작을 먼저 검증한 뒤, 동일한 결과를 코드만으로 재현할 수 있도록
Ansible 플레이북을 작성하고 클린 상태에서 재구축 검증까지 마쳤습니다.

## 아키텍처

```
                 ┌─────────────────────────────────────────┐
                 │   Host-only Network (192.168.56.0/24)    │
                 │                                           │
  ┌─────────┐    │  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
  │ client01│────┼──│  dns01  │  │  web01  │  │  db01   │   │
  │  .14    │    │  │  .11    │  │  .12    │  │  .13    │   │
  └─────────┘    │  └─────────┘  └─────────┘  └─────────┘   │
                 └─────────────────────────────────────────┘
     모든 VM은 NAT 네트워크로 외부(인터넷)와도 별도 연결됨
```

| 호스트 | IP | 역할 | 주요 서비스 |
|---|---|---|---|
| dns01 | 192.168.56.11 | 내부 DNS/NTP | BIND9 (lab.local 정방향/역방향 존), chrony |
| web01 | 192.168.56.12 | 웹 서버 | Nginx |
| db01 | 192.168.56.13 | DB 서버 | MariaDB |
| client01 | 192.168.56.14 | 테스트 클라이언트 | bind-utils, mariadb(client) |

- OS: Rocky Linux 9.8 Minimal
- 네트워크: Host-only(내부망) + NAT(외부 접속용) 이중 구성
- 모든 서버는 dns01을 nameserver로 사용하며, lab.local 도메인으로 서로를 조회

## 프로젝트 구조

```
lab-iac/
├── Vagrantfile
└── ansible/
    ├── site.yml
    ├── group_vars/
    │   └── db.yml.example      # 실제 db.yml은 git 추적 제외
    └── roles/
        ├── common/   # 패키지 업데이트, 내부 DNS 지정, 방화벽, 타임존
        ├── dns/       # BIND9, chrony
        ├── web/       # Nginx
        ├── db/        # MariaDB, 계정/보안 설정
        └── client/    # 테스트용 CLI 도구(dig, mysql)
```

- Vagrantfile의 `ansible.groups` 설정으로 각 VM은 `lab` 그룹과 자기 자신의 역할 그룹에만
  소속되어, `site.yml`에서 자신에게 해당하는 play만 실행합니다.
- Ansible은 `ansible_local` 방식을 사용합니다. 각 VM이 스스로 자신의 role만 실행합니다.

## 실행 방법

1. 저장소 클론 후 `ansible/group_vars/db.yml.example`을 복사해 `db.yml`을 만들고
   실제 값(DB 비밀번호 등)을 채웁니다.

   ```bash
   cp ansible/group_vars/db.yml.example ansible/group_vars/db.yml
   ```

2. VM을 순서대로 올립니다. (dns01을 가장 먼저 올려야 나머지 VM의 DNS 질의가 정상 동작합니다.)

   ```bash
   vagrant up dns01
   vagrant up web01
   vagrant up db01
   vagrant up client01
   ```

3. 검증(client01에서 크로스 체크)

   ```bash
   vagrant ssh client01
   dig +short web01.lab.local        # 192.168.56.12
   dig +short db01.lab.local         # 192.168.56.13
   dig +short -x 192.168.56.12       # web01.lab.local.
   curl -sI http://web01.lab.local   # HTTP/1.1 200 OK
   mysql -h db01.lab.local -u testuser -p -e "SELECT 1;"
   ```

## 트러블슈팅 로그

IaC 작업 과정에서 겪은 문제와 해결 과정입니다.

### 1. WSL2 환경에서 Vagrant VMware 플러그인 통신 실패

WSL2에서 Vagrant를 실행하면 Windows에 설치된 Vagrant VMware Utility 서비스(포트 9922)와
통신이 되지 않는 문제가 있었습니다. WSL 쪽에서는 해당 포트로 접근할 방법이 마땅치 않아,
Vagrant를 Windows 네이티브로 설치하는 방식으로 전환했습니다.

### 2. VMware 공유폴더(`/vagrant`) 자동 마운트 실패

기본 공유폴더 방식이 동작하지 않아, Ansible 코드를 실행하는 대신
`file` provisioner로 `ansible/` 디렉토리를 SSH를 통해 각 VM에 직접 업로드하는 방식을 사용했습니다.

### 3. Ansible pip 설치 시 저사양 VM에서 SSH 세션 끊김

`ansible_local` provisioner가 기본적으로 pip로 Ansible을 설치하려 했는데,
RAM이 부족한 VM에서 설치 도중 SSH 세션이 끊기는 문제가 발생했습니다.
`dnf install ansible-core`로 Ansible을 미리 설치해두는 방식으로 변경해 해결했습니다.

### 4. `dnf update` 이후 NetworkManager 버전 불일치

전체 패키지 업데이트 후 `nmcli`(클라이언트)와 실행 중인 `NetworkManager`(데몬)의
버전이 어긋나 이후 nmcli 관련 태스크가 실패했습니다.
`dnf update` 결과를 `register`로 받아, 변경이 있었을 때만 `NetworkManager` 서비스를
재시작하는 태스크를 추가해 해결했습니다.

### 5. DNS 설정이 프로필에는 반영되나 즉시 적용되지 않음

`nmcli` 모듈로 내부 DNS(dns01)를 연결 프로필에 지정했지만, 실행 중인 연결에는
곧바로 반영되지 않았습니다. 그 결과 client01에서 `lab.local` 도메인을 질의하면
내부 DNS 대신 NAT의 외부 DNS로 나가면서 엉뚱한 공인 IP를 응답받는 문제가 있었습니다.
DNS 설정 태스크의 결과를 `register`로 받아, 변경이 있었을 때만
`nmcli device reapply`로 연결을 즉시 재적용하도록 수정해 해결했습니다.

### 6. client01에 DB 클라이언트 도구 누락

처음에는 client role이 따로 없어 client01이 `common` role만 적용받고 있었고,
그 결과 `mysql`, `dig` 같은 테스트 도구가 설치되지 않은 상태였습니다.
`client` role을 새로 만들어 `bind-utils`, `mariadb`(클라이언트) 패키지를 설치하고,
`site.yml`에 해당 play를 추가해 해결했습니다.

### 7. db root 비밀번호 설정 태스크의 멱등성 문제

최초 실행 시에는 root 계정이 비밀번호 없이(소켓 인증) 접속되어 비밀번호 설정이
정상적으로 이루어졌지만, 이후 재실행부터는 이미 비밀번호가 설정된 root 계정에
자격 증명 없이 접속을 시도해 매번 실패했습니다.
해당 태스크에 `login_user`/`login_password`를 함께 지정하고
`check_implicit_admin: true`를 유지해, 비밀번호 유무와 관계없이
양쪽 상황에서 모두 동작하도록 수정했습니다.

## 검증

- 각 VM에 대해 `vagrant provision`을 두 번 연속 실행해 두 번째 실행에서
  `changed=0`이 나오는 것을 확인했습니다. (멱등성 검증)
- `vagrant destroy` 후 4대를 처음부터 다시 올려, 동일한 코드로 클린 상태에서도
  동일한 구성이 재현되는 것을 확인했습니다. (재현성 검증)
- client01에서의 DNS 질의(정방향/역방향), `curl`, DB 원격 접속까지
  전체 크로스 체크를 통과했습니다.

## 향후 계획

- FreeIPA를 이용한 중앙 인증 체계 추가
- Prometheus/Grafana를 이용한 모니터링 스택 추가
