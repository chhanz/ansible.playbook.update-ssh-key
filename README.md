# SSH Key 관리 Ansible Playbook

EC2 인스턴스의 SSH 키를 안전하게 일괄 교체하는 Ansible Playbook

## 📁 프로젝트 구조

```
manage_pem/
├── README.md                      # 프로젝트 문서 (본 파일)
├── ansible-update-ssh-key.yml     # SSH 키 관리 Playbook
└── inventory.ini                  # 대상 서버 및 변수 설정
```

## 🎯 주요 기능

### 3가지 작업 모드

1. **Deploy** - 새로운 SSH 키 배포 (기존 키 유지)
2. **Verify** - 새로운 키 배포 확인
3. **Cleanup** - 이전 SSH 키 제거

### 보안 기능

- ✅ 민감한 정보 로그 마스킹 (`no_log`)
- ✅ 자동 백업 (날짜 기반 파일명)
- ✅ 중복 실행 방지 (멱등성 보장)
- ✅ 키 존재 여부 검증
- ✅ 안전한 2단계 교체 프로세스

## 🚀 빠른 시작

### 1. 사전 요구사항

```bash
# Ansible 설치 확인
$ ansible --version
ansible [core 2.15.13]
...

# SSH 접근 확인
ssh -i private.pem ec2-user@<target-host>
```

### 2. 설정 파일 수정

**inventory.ini 편집**:
```ini
[all:vars]
target_user=ec2-user                          # 키를 변경할 사용자
new_private_key_path=test.pem                 # 새로운 Private Key 경로
ansible_ssh_private_key_file="private.pem"    # 현재 SSH 키
ansible_ssh_user="ec2-user"                   # Ansible 접속 사용자

[servers]
192.168.10.1
192.168.10.2
```

### 3. 실행

#### Step 1: 새 키 배포
```bash
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=deploy"
```

#### Step 2: 배포 확인
```bash
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=verify"
```

#### Step 3: 새 키로 접속 테스트
```bash
ansible servers -i inventory.ini -m ping --key-file test.pem
```

#### Step 4: 이전 키 제거
```bash
# inventory.ini에서 ansible_ssh_private_key_file을 test.pem으로 변경 후
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=cleanup"
```

## 📖 상세 사용법

### Deploy 모드

새로운 SSH 키를 추가합니다 (기존 키는 유지).

```bash
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=deploy"
```

**동작 과정**:
1. 사용자 존재 여부 확인
2. Private Key에서 Public Key 추출
3. authorized_keys에 키 존재 여부 확인
4. 기존 authorized_keys 백업
5. 새로운 키 추가

**생성되는 백업 파일**:
```
authorized_keys.backup.2026-02-03_051300
```

### Verify 모드

새로운 키가 정상적으로 배포되었는지 확인합니다.

```bash
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=verify"
```

**동작 과정**:
1. Private Key에서 Public Key 추출
2. authorized_keys에서 키 검색
3. 존재 여부 확인 및 결과 출력

**출력 예시**:
```
✓ VERIFIED: New SSH key is present in authorized_keys
```

### Cleanup 모드

이전 SSH 키를 제거합니다.

```bash
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini -e "operation_mode=cleanup"
```

**⚠️ 주의사항**:
- 반드시 새 키로 접속 가능한지 확인 후 실행

**동작 과정**:
1. 이전 Private Key에서 Public Key 추출
2. authorized_keys에서 키 존재 여부 확인
3. 백업 생성
4. 이전 키 제거
5. 제거 검증

**생성되는 백업 파일**:
```
authorized_keys.backup.cleanup_20260203_051430
```

## 🔧 고급 설정

### 특정 호스트만 실행

```bash
# 단일 호스트
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini \
  --limit 192.168.10.1 \
  -e "operation_mode=deploy"

# 여러 호스트
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini \
  --limit "192.168.10.1,192.168.10.2" \
  -e "operation_mode=deploy"
```

### 병렬 실행 수 조정

```bash
# 동시에 5대씩 처리
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini \
  -e "operation_mode=deploy" \
  -f 5
```

### Dry-run 모드

```bash
# 실제 변경 없이 시뮬레이션
ansible-playbook ansible-update-ssh-key.yml -i inventory.ini \
  -e "operation_mode=deploy" \
  --check
```

## 📊 백업 파일 관리

### 백업 파일 확인

```bash
# 모든 백업 파일 조회
ansible servers -i inventory.ini -m shell \
  -a "ls -lh ~/.ssh/authorized_keys.backup.*"

# 특정 날짜의 백업 파일
ansible servers -i inventory.ini -m shell \
  -a "ls -lh ~/.ssh/authorized_keys.backup.2026-02-03*"
```

### 백업 파일 정리

```bash
# 7일 이상 된 백업 파일 삭제
ansible servers -i inventory.ini -m shell \
  -a "find ~/.ssh -name 'authorized_keys.backup.*' -mtime +7 -delete"

# 최신 5개만 유지
ansible servers -i inventory.ini -m shell \
  -a "ls -t ~/.ssh/authorized_keys.backup.* | tail -n +6 | xargs rm -f"
```

### 백업 파일 복원

```bash
# 특정 백업으로 복원
ansible servers -i inventory.ini -m shell \
  -a "cp ~/.ssh/authorized_keys.backup.2026-02-03_051300 ~/.ssh/authorized_keys"
```
