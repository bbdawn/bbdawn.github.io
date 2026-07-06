---
layout: page
title: Octavia Manager
icon: fas fa-random
order: 2
permalink: /octavia-manager/
---

<style>
  #octavia-manager * { box-sizing: border-box; }
  #octavia-manager {
    margin: -0.5rem 0 0;
  }
  #octavia-manager .om-tabs {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
  }
  #octavia-manager .om-tab-btn {
    border: 1.5px solid var(--border-color, #dee2e6);
    background: var(--main-bg, #fff);
    color: var(--text-color, #333);
    border-radius: 8px;
    padding: 0.5rem 1rem;
    font-size: 0.88rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s;
  }
  #octavia-manager .om-tab-btn:hover { border-color: var(--link-color); }
  #octavia-manager .om-tab-btn.active {
    background: var(--link-color);
    border-color: var(--link-color);
    color: #fff;
  }

  #octavia-manager .om-panel { display: none; }
  #octavia-manager .om-panel.active { display: block; }

  #octavia-manager .om-card {
    border: 1px solid var(--border-color, #dee2e6);
    border-radius: 10px;
    padding: 1.25rem 1.4rem;
    margin-bottom: 1.25rem;
    background: var(--card-bg, rgba(0,0,0,0.015));
  }
  #octavia-manager .om-card h3 {
    font-size: 1.05rem;
    font-weight: 700;
    margin-bottom: 0.6rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  #octavia-manager .om-card h3 i { color: var(--link-color); opacity: 0.85; }
  #octavia-manager .om-card p { font-size: 0.92rem; line-height: 1.75; opacity: 0.85; }
  #octavia-manager .om-note {
    font-size: 0.85rem;
    padding: 0.7rem 1rem;
    border-left: 3px solid #dc3545;
    background: rgba(220,53,69,0.06);
    border-radius: 4px;
    margin: 0.75rem 0;
  }
  #octavia-manager .om-note.info {
    border-left-color: var(--link-color);
    background: rgba(var(--bs-primary-rgb, 13,110,253), 0.06);
  }

  #octavia-manager pre {
    overflow-x: auto;
    font-size: 0.82rem;
  }

  #octavia-manager table.om-table {
    width: 100%;
    font-size: 0.88rem;
    border-collapse: collapse;
    margin: 0.75rem 0 0.25rem;
  }
  #octavia-manager table.om-table th,
  #octavia-manager table.om-table td {
    border: 1px solid var(--border-color, #dee2e6);
    padding: 0.5rem 0.7rem;
    text-align: left;
  }
  #octavia-manager table.om-table th { background: rgba(0,0,0,0.03); }

  #octavia-manager .om-copy-wrap { position: relative; }
  #octavia-manager .om-copy-btn {
    position: absolute; top: 0.5rem; right: 0.5rem;
    font-size: 0.75rem; padding: 0.2rem 0.55rem;
    border: 1px solid var(--border-color, #dee2e6);
    background: var(--main-bg, #fff); border-radius: 5px; cursor: pointer;
    opacity: 0.7;
  }
  #octavia-manager .om-copy-btn:hover { opacity: 1; }

  #octavia-manager .om-step {
    display: flex; gap: 0.8rem; margin-bottom: 1rem;
  }
  #octavia-manager .om-step-num {
    flex-shrink: 0; width: 1.6rem; height: 1.6rem; border-radius: 50%;
    background: var(--link-color); color: #fff; font-size: 0.8rem; font-weight: 700;
    display: flex; align-items: center; justify-content: center;
  }
  #octavia-manager .om-step-body { font-size: 0.92rem; line-height: 1.7; }
  #octavia-manager .om-step-body code { font-size: 0.85rem; }

  #octavia-manager .om-id-form {
    display: flex; gap: 0.5rem; flex-wrap: wrap; margin-bottom: 1rem;
  }
  #octavia-manager .om-id-form input {
    flex: 1; min-width: 220px;
    border: 1.5px solid var(--border-color, #dee2e6);
    border-radius: 6px;
    padding: 0.45rem 0.75rem;
    font-size: 0.9rem;
    background: var(--main-bg, #fff);
    color: var(--text-color, #333);
  }
  #octavia-manager .om-id-form input:focus { outline: none; border-color: var(--link-color); }
  #octavia-manager .om-generated-badge {
    display: inline-block; font-size: 0.75rem; padding: 0.15rem 0.5rem;
    border-radius: 4px; background: rgba(var(--bs-primary-rgb, 13,110,253), 0.12);
    color: var(--link-color); margin-bottom: 0.5rem;
  }
</style>

<div id="octavia-manager">

  <div class="om-tabs">
    <button class="om-tab-btn active" data-panel="structure">구조</button>
    <button class="om-tab-btn" data-panel="cli">CLI / 로그</button>
    <button class="om-tab-btn" data-panel="cleanup">Pending 정리</button>
    <button class="om-tab-btn" data-panel="service">서비스 관리</button>
    <button class="om-tab-btn" data-panel="troubleshoot">트러블슈팅</button>
  </div>

  <!-- 구조 -->
  <div class="om-panel active" id="panel-structure">
    <div class="om-card">
      <h3><i class="fas fa-sitemap"></i> 리소스 계층 구조</h3>
      <pre><code>LoadBalancer  (VIP, Amphora &#215; 1~2 — active/standby)
  &#9500;&#9472;&#9472; Listener             (프로토콜/포트: HTTP, HTTPS, TCP, TERMINATED_HTTPS ...)
  &#9474;     &#9500;&#9472;&#9472; Pool (default_pool)
  &#9474;     &#9474;     &#9500;&#9472;&#9472; Pool Member &#215; N   (백엔드 서버 IP:Port, weight)
  &#9474;     &#9474;     &#9492;&#9472;&#9472; Health Monitor    (delay, timeout, max_retries, http_method ...)
  &#9474;     &#9492;&#9472;&#9472; L7 Policy &#215; N (선택)
  &#9474;           &#9500;&#9472;&#9472; L7 Rule &#215; N       (조건: path, header, cookie ...)
  &#9474;           &#9492;&#9472;&#9472; redirect &#8594; 다른 Pool
  &#9500;&#9472;&#9472; VIP                  (Virtual IP, 실제 트래픽이 들어오는 주소)
  &#9492;&#9472;&#9472; VRRP Group           (Amphora 이중화용 메타데이터, HA 구성 시)</code></pre>

      <p style="margin-top:0.9rem">하나의 LoadBalancer는 여러 Listener를 가질 수 있고, 각 Listener는 기본 Pool(default_pool) 하나와 선택적으로 여러 L7 Policy를 가집니다. 실제로는 이 모든 논리 리소스가 <strong>Amphora</strong>라는 VM(Nova 인스턴스) 위에서 동작하며, VIP·VRRP Group은 이 Amphora의 이중화·네트워크 정보를 담습니다.</p>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-layer-group"></i> 리소스별 역할</h3>
      <table class="om-table">
        <tr><th>리소스</th><th>역할</th></tr>
        <tr>
          <td><code>LoadBalancer</code></td>
          <td>전체 로드밸런싱 서비스의 최상위 객체. VIP(가상 IP)를 가지며, 실제로는 Amphora(Nova VM) 위에서 동작합니다. 생성 시 Amphora가 배포되고, 이후 모든 하위 리소스(Listener, Pool 등)가 여기 종속됩니다.</td>
        </tr>
        <tr>
          <td><code>Listener</code></td>
          <td>트래픽을 수신하는 지점. 프로토콜(HTTP/HTTPS/TCP/TERMINATED_HTTPS)과 포트를 정의합니다. 하나의 LB에 여러 Listener를 둘 수 있어(예: 80, 443) 서로 다른 프로토콜/포트의 트래픽을 각각 처리할 수 있습니다.</td>
        </tr>
        <tr>
          <td><code>Pool</code></td>
          <td>실제 트래픽을 분산시킬 백엔드 서버 그룹. 로드밸런싱 알고리즘(ROUND_ROBIN, LEAST_CONNECTIONS, SOURCE_IP 등)을 지정합니다. Listener의 default_pool로 바로 연결되거나, L7 Policy를 통해 조건부로 연결됩니다.</td>
        </tr>
        <tr>
          <td><code>Pool Member</code></td>
          <td>Pool에 속한 개별 백엔드 서버. IP:Port와 가중치(weight)를 가지며, weight가 높을수록 더 많은 트래픽을 받습니다. Health Monitor 결과에 따라 개별적으로 활성/비활성 상태가 바뀝니다.</td>
        </tr>
        <tr>
          <td><code>Health Monitor</code></td>
          <td>Pool Member의 상태를 주기적으로 점검해서 비정상 서버를 트래픽 분배 대상에서 자동으로 제외합니다. HTTP/TCP/PING 등 방식으로 체크하며, 연속 실패 횟수(<code>max_retries_down</code>)를 넘기면 <code>OFFLINE</code>으로 표시됩니다.</td>
        </tr>
      </table>
    </div>
  </div>

  <!-- CLI / 로그 -->
  <div class="om-panel" id="panel-cli">
    <div class="om-card">
      <h3><i class="fas fa-terminal"></i> OpenStack CLI 사용 방법</h3>
      <p>인증 정보(openrc)를 먼저 불러온 뒤 명령어를 사용합니다.</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>source contrabass-openrc

# 로드밸런서 목록 조회
openstack loadbalancer list</code></pre>
      </div>
      <p style="margin-top:0.9rem">조회 결과 예시 (값은 예시로 대체):</p>
      <pre><code>+--------------------------------------+----------------+----------------------------------+-------------+---------------------+-------------------+----------+
| id                                   | name           | project_id                       | vip_address | provisioning_status | operating_status  | provider |
+--------------------------------------+----------------+----------------------------------+-------------+---------------------+-------------------+----------+
| 3291b02d-2ebd-4d50-9c5d-8b8139eae21b | web-lb-01      | 11111111111111111111111111111111 | 10.0.0.10   | ACTIVE              | ONLINE            | amphora  |
| 91470ce0-f690-44d7-b26c-788fa1d4de89 | stuck-test-lb  | 22222222222222222222222222222222 | 10.0.0.11   | PENDING_DELETE      | OFFLINE           | amphora  |
| c6a4a7fb-f219-4036-9c90-e96cfcc13722 | api-lb-02      | 11111111111111111111111111111111 | 10.0.0.12   | ERROR               | OFFLINE           | amphora  |
+--------------------------------------+----------------+----------------------------------+-------------+---------------------+-------------------+----------+</code></pre>
      <p>확인해야 할 핵심 컬럼은 <code>provisioning_status</code>(생성/삭제 등 작업 진행 상태)와 <code>operating_status</code>(실제 트래픽 처리 가능 여부)입니다.</p>

      <p style="margin-top:1rem"><strong>provisioning_status</strong> — 작업 진행 상태</p>
      <table class="om-table">
        <tr><th>값</th><th>의미</th></tr>
        <tr><td><code>ACTIVE</code></td><td>정상적으로 생성/수정 완료된 상태</td></tr>
        <tr><td><code>PENDING_CREATE</code></td><td>생성 작업 진행 중</td></tr>
        <tr><td><code>PENDING_UPDATE</code></td><td>수정 작업 진행 중</td></tr>
        <tr><td><code>PENDING_DELETE</code></td><td>삭제 작업 진행 중</td></tr>
        <tr><td><code>ERROR</code></td><td>작업 실패 (재시도하거나 정리 필요)</td></tr>
      </table>

      <p style="margin-top:1rem"><strong>operating_status</strong> — 실제 동작 상태</p>
      <table class="om-table">
        <tr><th>값</th><th>의미</th></tr>
        <tr><td><code>ONLINE</code></td><td>정상 동작 중, 트래픽 처리 가능</td></tr>
        <tr><td><code>OFFLINE</code></td><td>관리자에 의해 비활성화됨(admin_state_up=false) 또는 헬스체크 이전</td></tr>
        <tr><td><code>DEGRADED</code></td><td>일부 Pool Member만 정상 (예: 3대 중 1대만 UP)</td></tr>
        <tr><td><code>ERROR</code></td><td>심각한 오류로 트래픽 처리 불가</td></tr>
        <tr><td><code>DRAINING</code></td><td>Pool Member 대상 — 기존 연결은 유지하되 신규 연결은 받지 않는 종료 대기 상태</td></tr>
        <tr><td><code>NO_MONITOR</code></td><td>Health Monitor가 설정되지 않아 상태를 판단할 수 없음</td></tr>
      </table>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-list"></i> 리소스별 조회 명령어</h3>
      <table class="om-table">
        <tr><th>리소스</th><th>명령어</th></tr>
        <tr><td>LoadBalancer</td><td><code>openstack loadbalancer list</code> / <code>show &lt;lb-id&gt;</code></td></tr>
        <tr><td>Listener</td><td><code>openstack loadbalancer listener list</code> / <code>show &lt;listener-id&gt;</code></td></tr>
        <tr><td>Pool</td><td><code>openstack loadbalancer pool list</code> / <code>show &lt;pool-id&gt;</code></td></tr>
        <tr><td>Pool Member</td><td><code>openstack loadbalancer member list &lt;pool-id&gt;</code> / <code>show &lt;pool-id&gt; &lt;member-id&gt;</code></td></tr>
        <tr><td>Health Monitor</td><td><code>openstack loadbalancer healthmonitor list</code> / <code>show &lt;hm-id&gt;</code></td></tr>
        <tr><td>L7 Policy</td><td><code>openstack loadbalancer l7policy list</code> / <code>show &lt;policy-id&gt;</code></td></tr>
        <tr><td>L7 Rule</td><td><code>openstack loadbalancer l7rule list &lt;policy-id&gt;</code> / <code>show &lt;policy-id&gt; &lt;rule-id&gt;</code></td></tr>
        <tr><td>Amphora</td><td><code>openstack loadbalancer amphora list</code> / <code>show &lt;amphora-id&gt;</code></td></tr>
      </table>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-hammer"></i> 생성 흐름 (전체 구성 예시)</h3>
      <p>구조 탭의 계층과 동일한 순서로 생성합니다. LoadBalancer → Listener → Pool → Pool Member → Health Monitor.</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code># 1. LoadBalancer 생성
openstack loadbalancer create --name my-lb --vip-subnet-id &lt;subnet-id&gt;

# 2. Listener 생성
openstack loadbalancer listener create --name my-listener \
  --protocol HTTP --protocol-port 80 &lt;lb-id&gt;

# 3. Pool 생성
openstack loadbalancer pool create --name my-pool \
  --lb-algorithm ROUND_ROBIN --listener &lt;listener-id&gt; --protocol HTTP

# 4. Pool Member 추가
openstack loadbalancer member create --address 10.0.0.21 \
  --protocol-port 8080 &lt;pool-id&gt;

# 5. Health Monitor 생성
openstack loadbalancer healthmonitor create --delay 5 --timeout 3 \
  --max-retries 3 --type HTTP &lt;pool-id&gt;</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-tools"></i> 운영에 자주 쓰는 명령어</h3>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code># LB 트래픽/통계 확인
openstack loadbalancer stats show &lt;lb-id&gt;

# Amphora 강제 failover (장애 시 수동 복구)
openstack loadbalancer amphora failover &lt;amphora-id&gt;

# LoadBalancer 강제 삭제 (하위 리소스까지 한번에)
openstack loadbalancer delete --cascade &lt;lb-id&gt;

# 쿼터 확인
openstack loadbalancer quota show &lt;project-id&gt;</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-file-alt"></i> Octavia 로그 확인</h3>
      <p>Octavia 각 서비스의 로그는 <code>/var/log/octavia/</code>에 서비스별로 분리되어 쌓입니다.</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>cd /var/log/octavia
ls -la
# octavia-api.log
# octavia-worker.log
# octavia-health-manager.log
# octavia-housekeeping.log

# 자주 보는 로그
tail -f octavia-api.log
tail -f octavia-worker.log</code></pre>
      </div>
      <p>대부분의 생성/삭제/상태전이 관련 이슈는 <code>octavia-worker.log</code>에서 흔적을 찾을 수 있습니다.</p>
    </div>
  </div>

  <!-- Pending 정리 -->
  <div class="om-panel" id="panel-cleanup">
    <div class="om-note info">
      아래 절차는 순서대로 진행합니다: ① PENDING_* 상태를 ERROR로 전환 → ② 포탈/CLI로 cascade 삭제 시도 → ③ 그래도 안 지워지면 DB 직접 삭제(최후 수단).
    </div>

    <div class="om-card">
      <h3><i class="fas fa-exchange-alt"></i> ① PENDING_* 상태를 ERROR로 전환</h3>
      <p>생성/수정/삭제 도중 멈춰서 <code>PENDING_CREATE</code> / <code>PENDING_UPDATE</code> / <code>PENDING_DELETE</code>에 계속 머물러 있는 LB는, 상태를 <code>ERROR</code>로 바꿔줘야 이후 삭제가 가능합니다.</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>mysql -u octavia -p
# 비밀번호 입력

use octavia;

UPDATE load_balancer
SET provisioning_status = 'ERROR'
WHERE provisioning_status IN (
    'PENDING_CREATE',
    'PENDING_UPDATE',
    'PENDING_DELETE'
);

-- 확인
SELECT id, name, operating_status, provisioning_status
FROM load_balancer
WHERE provisioning_status != 'DELETED';</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-trash-alt"></i> ② 포탈 / Horizon / CLI로 삭제</h3>
      <p>상태가 <code>ERROR</code>로 바뀌면 콘트라베이스 포탈이나 Horizon에서 삭제하거나, CLI로 cascade 삭제합니다. LB ID를 입력하면 아래 명령어가 자동으로 채워집니다.</p>

      <div class="om-id-form">
        <input id="om-lb-id" type="text" placeholder="로드밸런서 ID 입력 (예: 91470ce0-f690-44d7-b26c-788fa1d4de89)" oninput="omRenderAll()">
      </div>

      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code id="om-cascade-cmd">openstack loadbalancer delete --cascade &lt;lb-id&gt;</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-search"></i> ③ 삭제 전 확인 — 대상 리소스 조회</h3>
      <p>DB를 직접 삭제하기 전에, 이 LB에 딸린 리소스가 실제로 몇 개나 있는지 먼저 조회해서 확인합니다.</p>
      <span class="om-generated-badge" id="om-select-badge">위 입력창의 ID로 자동 생성됨 (미입력 시 예시 ID 사용)</span>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code id="om-select-sql"></code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-database"></i> ④ 그래도 안 지워지면: DB 직접 삭제</h3>
      <div class="om-note">
        최후 수단입니다. 삭제 순서를 반드시 지켜야 참조 무결성 오류가 나지 않습니다. 실행 전 반드시 백업하고, 위 ③ 조회 결과로 대상이 맞는지 먼저 확인하세요.
      </div>
      <span class="om-generated-badge" id="om-sql-badge">위 입력창의 ID로 자동 생성됨 (미입력 시 예시 ID 사용)</span>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code id="om-delete-sql"></code></pre>
      </div>
    </div>
  </div>

  <!-- 서비스 관리 -->
  <div class="om-panel" id="panel-service">
    <div class="om-card">
      <h3><i class="fas fa-server"></i> Octavia 서비스 구성 (systemd)</h3>
      <table class="om-table">
        <tr><th>서비스</th><th>역할</th></tr>
        <tr><td><code>octavia-api</code></td><td>REST API 엔드포인트 — LB/Listener/Pool 등 CRUD 요청 수신</td></tr>
        <tr><td><code>octavia-worker</code></td><td>실제 생성/수정/삭제 작업 처리 (Amphora 배포 포함)</td></tr>
        <tr><td><code>octavia-health-manager</code></td><td>Amphora 헬스체크 및 장애 감지, failover 트리거</td></tr>
        <tr><td><code>octavia-housekeeping</code></td><td>오래된/미사용 Amphora 및 리소스 정리</td></tr>
        <tr><td><code>octavia-interface</code></td><td>Amphora 통신용 네트워크 인터페이스(o-hm0) 생성 (1회성 실행 후 exited 상태가 정상)</td></tr>
      </table>
      <div class="om-copy-wrap" style="margin-top:0.75rem">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>systemctl list-units | grep octavia</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-terminal"></i> 상태 확인</h3>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>systemctl status octavia-api
systemctl status octavia-worker
systemctl status octavia-health-manager
systemctl status octavia-housekeeping
systemctl status octavia-interface</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-redo"></i> 재시작</h3>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>systemctl restart octavia-worker
systemctl status octavia-worker

systemctl restart octavia-health-manager
systemctl status octavia-health-manager

systemctl restart octavia-housekeeping
systemctl status octavia-housekeeping

systemctl restart octavia-api
systemctl status octavia-api</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-network-wired"></i> o-hm0 인터페이스 확인</h3>
      <p>Octavia가 Amphora와 통신하는 데 쓰는 헬스매니저 전용 인터페이스입니다. 이게 없으면 LB 생성 자체가 실패합니다.</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>ip a | grep o-hm0</code></pre>
      </div>
    </div>
  </div>

  <!-- 트러블슈팅 -->
  <div class="om-panel" id="panel-troubleshoot">
    <div class="om-card">
      <h3><i class="fas fa-stethoscope"></i> 기본 진단 흐름</h3>

      <div class="om-step">
        <div class="om-step-num">1</div>
        <div class="om-step-body">LB 상태 확인<br><code>openstack loadbalancer show &lt;lb-id&gt;</code> — <code>provisioning_status</code>, <code>operating_status</code> 확인</div>
      </div>
      <div class="om-step">
        <div class="om-step-num">2</div>
        <div class="om-step-body">Amphora 상태 확인<br><code>openstack loadbalancer amphora list --loadbalancer &lt;lb-id&gt;</code></div>
      </div>
      <div class="om-step">
        <div class="om-step-num">3</div>
        <div class="om-step-body">서비스 로그 확인<br><code>tail -f /var/log/octavia/octavia-worker.log</code></div>
      </div>
      <div class="om-step">
        <div class="om-step-num">4</div>
        <div class="om-step-body">그래도 안 풀리면 증상별 케이스로 이동 (아래)</div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-bug"></i> 자주 발생하는 케이스</h3>
      <table class="om-table">
        <tr><th>증상</th><th>참고</th></tr>
        <tr>
          <td><code>PENDING_*</code>에서 멈춘 LB</td>
          <td>"Pending 정리" 탭 참고</td>
        </tr>
        <tr>
          <td>o-hm0 인터페이스 없어서 생성 실패</td>
          <td><a href="{% post_url 2026-07-01-octavia-o-hm0-missing %}">Loadbalancer 생성 실패 원인(2)</a></td>
        </tr>
        <tr>
          <td>Anti-affinity policy 위반으로 생성 실패</td>
          <td><a href="{% post_url 2026-06-30-openstack-anti-affinity-violation %}">Loadbalancer 생성 실패 원인(1)</a></td>
        </tr>
      </table>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-info-circle"></i> 참고 — 쿼터</h3>
      <p>Amphora VM은 내부적으로 <code>service</code> 프로젝트에 생성되므로, LB를 대량으로 생성/삭제할 때는 <code>service</code> 프로젝트의 <strong>서버 그룹(Server Group) 쿼터</strong>와 <strong>보안 그룹(Security Group) 쿼터</strong>도 함께 확인해야 합니다.</p>
    </div>
  </div>

</div>

<script>
(function () {
  document.querySelectorAll('#octavia-manager .om-tab-btn').forEach(function (btn) {
    btn.addEventListener('click', function () {
      document.querySelectorAll('#octavia-manager .om-tab-btn').forEach(function (b) { b.classList.remove('active'); });
      document.querySelectorAll('#octavia-manager .om-panel').forEach(function (p) { p.classList.remove('active'); });
      btn.classList.add('active');
      document.getElementById('panel-' + btn.dataset.panel).classList.add('active');
    });
  });
  omRenderAll();
})();

function omCopy(btn) {
  var pre = btn.parentElement.querySelector('pre');
  var text = pre.innerText;
  navigator.clipboard.writeText(text).then(function () {
    var orig = btn.textContent;
    btn.textContent = '복사됨';
    setTimeout(function () { btn.textContent = orig; }, 1200);
  });
}

function omRenderAll() {
  var input = document.getElementById('om-lb-id');
  var raw = input ? input.value.trim() : '';
  var id = raw || '91470ce0-f690-44d7-b26c-788fa1d4de89';
  var badgeText = raw
    ? '입력하신 ID(' + raw + ')로 생성됨'
    : '위 입력창의 ID로 자동 생성됨 (미입력 시 예시 ID 사용)';
  ['om-sql-badge', 'om-select-badge'].forEach(function (bid) {
    var b = document.getElementById(bid);
    if (b) b.textContent = badgeText;
  });

  var selectEl = document.getElementById('om-select-sql');
  if (selectEl) {
    selectEl.textContent =
'SELECT * FROM load_balancer WHERE id = \'' + id + '\';\n\n' +
'SELECT * FROM listener WHERE load_balancer_id = \'' + id + '\';\n\n' +
'SELECT * FROM pool WHERE load_balancer_id = \'' + id + '\';\n\n' +
'SELECT * FROM member\n' +
'WHERE pool_id IN (SELECT id FROM pool WHERE load_balancer_id = \'' + id + '\');\n\n' +
'SELECT * FROM health_monitor\n' +
'WHERE pool_id IN (SELECT id FROM pool WHERE load_balancer_id = \'' + id + '\');\n\n' +
'SELECT * FROM amphora WHERE load_balancer_id = \'' + id + '\';\n\n' +
'SELECT * FROM vip WHERE load_balancer_id = \'' + id + '\';\n\n' +
'SELECT * FROM vrrp_group WHERE load_balancer_id = \'' + id + '\';';
  }

  var cascadeEl = document.getElementById('om-cascade-cmd');
  if (cascadeEl) {
    cascadeEl.textContent = 'openstack loadbalancer delete --cascade ' + id;
  }

  var sqlEl = document.getElementById('om-delete-sql');
  if (sqlEl) {
    sqlEl.textContent =
'-- 1. Health Monitor 삭제\n' +
'DELETE FROM health_monitor\n' +
'WHERE pool_id IN (\n' +
'    SELECT id FROM pool\n' +
"    WHERE load_balancer_id = '" + id + "'\n" +
');\n\n' +
'-- 2. Pool Member 삭제\n' +
'DELETE FROM member\n' +
'WHERE pool_id IN (\n' +
'    SELECT id FROM pool\n' +
"    WHERE load_balancer_id = '" + id + "'\n" +
');\n\n' +
'-- 3. Listener -> default_pool_id 참조 끊기\n' +
'UPDATE listener\n' +
"SET default_pool_id = NULL\n" +
'WHERE default_pool_id IN (\n' +
'    SELECT id FROM pool\n' +
"    WHERE load_balancer_id = '" + id + "'\n" +
');\n\n' +
'-- 4. Pool 삭제\n' +
'DELETE FROM pool\n' +
"WHERE load_balancer_id = '" + id + "';\n\n" +
'-- 5. Listener 삭제\n' +
'DELETE FROM listener\n' +
"WHERE load_balancer_id = '" + id + "';\n\n" +
'-- 6. amphora_health 삭제\n' +
'DELETE FROM amphora_health\n' +
'WHERE amphora_id IN (\n' +
'    SELECT id FROM amphora\n' +
"    WHERE load_balancer_id = '" + id + "'\n" +
');\n\n' +
'-- 7. amphora 삭제\n' +
'DELETE FROM amphora\n' +
"WHERE load_balancer_id = '" + id + "';\n\n" +
'-- 8. VIP 삭제\n' +
'DELETE FROM vip\n' +
"WHERE load_balancer_id = '" + id + "';\n\n" +
'-- 9. vrrp_group 삭제 (Amphora HA 메타 테이블)\n' +
'DELETE FROM vrrp_group\n' +
"WHERE load_balancer_id = '" + id + "';\n\n" +
'-- 10. LoadBalancer 삭제\n' +
'DELETE FROM load_balancer\n' +
"WHERE id = '" + id + "';";
  }
}
</script>
