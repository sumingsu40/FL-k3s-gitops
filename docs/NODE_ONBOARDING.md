# 새 노드 추가 가이드

이 문서는 FL k3s 클러스터에 새 worker 노드(주로 Jetson)를 추가할 때 필요한 모든 설정을 정리한다.

## 왜 이런 절차가 필요한가

- Control-plane(`sdnserver01`)과 worker 노드들은 **서로 다른 네트워크**에 있다.
  - sdnserver01: 공인 IP `210.94.193.127` / `210.94.193.125`
  - jetson 노드들: 사설 LAN `192.168.50.0/24` (라우터 NAT 뒤, 같은 공인 IP 공유)
- k3s flannel VXLAN 터널은 노드의 `internal-ip`로 양방향 통신해야 하는데, 이 IP들이 서로 라우팅되지 않는다.
- **모든 노드를 Tailscale mesh의 `100.x.x.x` IP로 통일**해서 k3s가 그 위로 통신하도록 한다. Tailscale이 NAT/방화벽을 자동 우회하고 트래픽을 암호화한다.

따라서 **새 노드는 반드시 Tailscale에 가입되어 있어야 한다**.

---

## 사전 요구사항

| 항목 | 비고 |
|---|---|
| Ubuntu 22.04+ | Jetson은 JetPack 기본 |
| 인터넷 접속 | k3s 설치, Tailscale 인증, 이미지 풀 |
| sudo 권한 | k3s 설치 및 systemd 등록 |
| (Jetson만) NVIDIA Container Runtime | `runtimeClassName: nvidia`로 GPU 사용 |

---

## 절차

### 1. Tailscale 설치 및 가입

새 노드에서 실행:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# 출력된 URL을 브라우저에서 열어 isnlab.manager 계정으로 인증
```

가입 확인 (control-plane에서):

```bash
tailscale status | grep <새노드명>
```

→ `100.x.x.x` IP가 출력되어야 한다. 이 IP를 **반드시 메모**(`<NEW_TS_IP>`).

### 2. k3s 서버 토큰 확인 (control-plane에서)

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

출력값이 `<K3S_TOKEN>`. 새 노드의 K3S_TOKEN 환경변수로 들어간다.

### 3. (Jetson인 경우) NVIDIA runtime 설치 확인

```bash
# 새 노드에서
ls /usr/bin/nvidia-container-runtime || \
  sudo apt-get install -y nvidia-container-toolkit
```

### 4. k3s-agent 설치 (새 노드에서)

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://210.94.193.125:6443 \
  K3S_TOKEN=<K3S_TOKEN> \
  INSTALL_K3S_EXEC="agent" \
  sh -
```

**아직 자동 시작은 막을 것** — config.yaml 작성 전에 시작되면 잘못된 IP로 등록된다. install 스크립트가 자동 시작하면 즉시 stop:

```bash
sudo systemctl stop k3s-agent
```

### 5. config.yaml 작성 (새 노드에서)

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
node-ip: <NEW_TS_IP>
node-external-ip: <NEW_TS_IP>
flannel-iface: tailscale0
EOF
```

`<NEW_TS_IP>`를 1단계에서 메모한 Tailscale IP로 치환한다.

### 6. k3s-agent 시작

```bash
sudo systemctl enable --now k3s-agent
sudo systemctl status k3s-agent --no-pager | head -10
```

### 7. 노드 등록 확인 (control-plane에서)

```bash
kubectl get nodes -o wide
```

새 노드가 `Ready`이고 INTERNAL-IP가 `100.x.x.x`여야 한다. 사설 IP(`192.168.x.x`)나 공인 IP가 보이면 실패 — config.yaml 다시 확인 후 `sudo systemctl restart k3s-agent`.

### 8. CID 등록 (control-plane에서)

새 클라이언트마다 **고유 CID**가 필요하다. CID 매핑은 `manifests/clients/fl-client-id-map.yaml`의 ConfigMap 한 곳에서 관리한다. 새 노드는 다음 빈 번호를 받는다.

```yaml
# manifests/clients/fl-client-id-map.yaml
data:
  ids: |
    jetson02=0
    jetson03=1
    jetson04=2
    jetson01=3
    yhgwpi=4
    <새노드명>=5   # ← 추가
```

그리고 서버 `manifests/server/fl-server.yaml`의 `-nc` 값을 총 클라이언트 수로 올린다.

```yaml
- "-nc"
- "6"   # ← 기존 5에서 1 증가
```

### 9. 노드 라벨 부여 (control-plane에서)

DaemonSet별로 다른 라벨을 쓴다. 노드의 디바이스 종류와 사용할 이미지에 따라 선택:

| 노드 종류 | 라벨 | 매칭되는 DaemonSet | 이미지 |
|---|---|---|---|
| jetson02~04 (orin) | `device=jetson` | `fl-client-jetson` | `fl-client-orin:v2.2` |
| jetson01 | `device=jetson01` | `fl-client-jetson01` | `fl-client-jetson:v2.3` |
| Raspberry Pi | `device=rpi` | `fl-client-rpi` | `fl-client-rpi:v2.3` |

```bash
kubectl label node <새노드명> device=<rpi|jetson|jetson01>
```

이미지가 같은 device class에 새 노드를 붙이면 라벨링만으로 자동 배포. **새 image variant가 필요하면 별도 DaemonSet 매니페스트를 추가**한다(`fl-client-jetson01.yaml` 참조).

### 10. 통신 검증

```bash
kubectl logs -n fl-system -l app=fl-client --tail=20
```

`Connection failed` 메시지가 안 나오고 학습/다운로드 로그가 흐르면 성공. CID 매핑이 빠진 경우 `no CID mapping for node: <노드명>`이 출력된다 — 8단계로 돌아가 ConfigMap을 확인한다.

---

## 노드 제거

```bash
# control-plane에서
kubectl drain <노드명> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <노드명>
```

새 노드의 k3s-agent를 멈추지 않으면 자동 재등록되므로, 새 노드에서:

```bash
sudo systemctl stop k3s-agent
sudo systemctl disable k3s-agent
```

agent 바이너리까지 완전 제거하려면 `sudo /usr/local/bin/k3s-agent-uninstall.sh`.

---

## 트러블슈팅

### `kubectl get nodes`에 사설 IP가 보임

`/etc/rancher/k3s/config.yaml`이 안 읽혔거나 잘못 작성됨. 확인:

```bash
sudo cat /etc/rancher/k3s/config.yaml
sudo systemctl restart k3s-agent
```

### Pod이 `Connection failed: Retrying...`만 반복

flannel 라우트가 형성 안 됨. control-plane에서:

```bash
ip route | grep 10.42  # 모든 노드 CIDR(10.42.X.0/24)이 flannel.1 via로 보여야 함
```

라우트가 빠진 노드가 있으면 그 노드에서 `sudo systemctl restart k3s-agent`.

### `tailscale0` 인터페이스 없음

```bash
sudo tailscale up
ip link show tailscale0
```

### 인증서 SAN 오류

새 IP에서 API 접근 시 `x509: certificate is valid for ...` 오류 발생 시, control-plane에서 `/etc/rancher/k3s/config.yaml`의 `tls-san:` 항목에 추가 후 `sudo systemctl restart k3s`.

---

## 부록: 현재 클러스터 노드

| 노드 | CID | 역할 | Tailscale IP | SSH |
|---|---|---|---|---|
| sdnserver01 | — | control-plane | `100.95.142.115` | `user@210.94.193.125 -p 22` |
| jetson02 | 0 | worker (jetson, orin v2.2) | `100.97.235.111` | `jetson02@210.94.189.113 -p 1032` |
| jetson03 | 1 | worker (jetson, orin v2.2) | `100.115.24.101` | `jetson03@210.94.189.113 -p 1033` |
| jetson04 | 2 | worker (jetson, orin v2.2) | `100.98.62.26` | `jetson04@210.94.189.113 -p 1034` |
| jetson01 | 3 | worker (jetson v2.3) | _TBD_ | _TBD_ |
| yhgwpi | 4 | worker (rpi v2.3, CPU) | _TBD_ | _TBD_ |

---

## 부록: 참고 매니페스트

워크로드 매니페스트는 `manifests/clients/`에 있다:

| 파일 | 대상 노드 | 비고 |
|---|---|---|
| `fl-client-id-map.yaml` | (전체 공용) | 노드→CID ConfigMap + entrypoint 스크립트 — **CID는 여기 한 곳만** |
| `fl-client-jetson.yaml` | jetson02~04 | orin v2.2, nvidia runtime, `-d cuda` |
| `fl-client-jetson01.yaml` | jetson01 | jetson v2.3, nvidia runtime, `-d cuda` |
| `fl-client-rpi.yaml` | yhgwpi (+ 추후 rpi들) | rpi v2.3, runtime 없음, `-d cpu` |

운영 원칙:
- 모든 DaemonSet은 `fl-client-id-map`의 `entrypoint.sh`를 공유한다 → 노드명만 알면 CID 자동 매핑
- 같은 이미지/device class에 새 노드 추가 시: ConfigMap에 `노드명=N` 한 줄 + 노드 라벨 + 서버 `-nc` 증가
- **새 이미지 변형이 필요한 노드**: 새 DaemonSet 파일을 만들고 nodeSelector 라벨도 새로 정의 (`fl-client-jetson01.yaml` 참조)
- 변경은 git push → ArgoCD 자동 sync
