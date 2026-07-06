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

  #octavia-manager .om-cmd-list {
    display: flex; flex-direction: column; gap: 0.4rem;
    margin: 0.75rem 0 0.25rem;
  }
  #octavia-manager .om-cmd-row {
    display: flex; align-items: center; gap: 0.6rem;
    background: var(--code-bg, #f6f6f6);
    border-radius: 6px;
    padding: 0.45rem 0.5rem 0.45rem 0.75rem;
  }
  #octavia-manager .om-cmd-label {
    flex-shrink: 0; min-width: 92px;
    font-size: 0.78rem; opacity: 0.6;
  }
  #octavia-manager .om-cmd-row code {
    flex: 1; min-width: 0;
    background: none; padding: 0;
    font-size: 0.82rem;
    white-space: pre; overflow-x: auto;
  }
  #octavia-manager .om-cmd-copy {
    flex-shrink: 0; border: none; background: none; cursor: pointer;
    color: var(--text-muted-color, #6c757d);
    padding: 0.25rem 0.45rem; border-radius: 5px;
    font-size: 0.85rem; transition: all 0.15s;
  }
  #octavia-manager .om-cmd-copy:hover { color: var(--link-color); background: rgba(0,0,0,0.06); }
  #octavia-manager .om-cmd-copy.copied { color: #28a745; }

  #octavia-manager .om-step-body .om-cmd-copy {
    margin-left: 0.35rem;
  }

  #octavia-manager table.om-table .om-cmd-copy {
    margin-left: 0.3rem;
  }

  #octavia-manager .om-tree { display: flex; flex-direction: column; gap: 0.55rem; }
  #octavia-manager .om-tree-node {
    display: flex; align-items: flex-start; gap: 0.75rem;
    background: var(--code-bg, #f6f6f6);
    border-left: 4px solid var(--om-lvl-color, #3b82f6);
    border-radius: 8px;
    padding: 0.65rem 0.9rem;
  }
  #octavia-manager .om-tree-lvl0 { --om-lvl-color: #3b82f6; margin-left: 0; }
  #octavia-manager .om-tree-lvl0 .om-tree-icon { background: rgba(59,130,246,0.15); color: #3b82f6; }
  #octavia-manager .om-tree-lvl1 { --om-lvl-color: #10b981; margin-left: 1.5rem; }
  #octavia-manager .om-tree-lvl1 .om-tree-icon { background: rgba(16,185,129,0.15); color: #10b981; }
  #octavia-manager .om-tree-lvl2 { --om-lvl-color: #f59e0b; margin-left: 3rem; }
  #octavia-manager .om-tree-lvl2 .om-tree-icon { background: rgba(245,158,11,0.15); color: #f59e0b; }
  #octavia-manager .om-tree-lvl3 { --om-lvl-color: #8b5cf6; margin-left: 4.5rem; }
  #octavia-manager .om-tree-lvl3 .om-tree-icon { background: rgba(139,92,246,0.15); color: #8b5cf6; }
  #octavia-manager .om-tree-icon {
    flex-shrink: 0; width: 2rem; height: 2rem; border-radius: 50%;
    display: flex; align-items: center; justify-content: center;
    font-size: 0.9rem;
  }
  #octavia-manager .om-tree-title {
    font-weight: 700; font-size: 0.95rem;
    display: flex; align-items: center; gap: 0.5rem; flex-wrap: wrap;
  }
  #octavia-manager .om-tree-tag {
    font-size: 0.72rem; font-weight: 600; opacity: 0.7;
    background: rgba(0,0,0,0.06); border-radius: 4px; padding: 0.1rem 0.4rem;
  }
  #octavia-manager .om-tree-desc { font-size: 0.85rem; opacity: 0.75; line-height: 1.6; margin-top: 0.15rem; }

  #octavia-manager .om-ha-diagram {
    font-family: var(--font-monospace, monospace);
    font-size: 0.8rem; line-height: 1.85;
  }

  #octavia-manager .om-sql-step { margin-bottom: 1rem; }
  #octavia-manager .om-sql-step:last-child { margin-bottom: 0; }
  #octavia-manager .om-sql-step .om-cmd-label { min-width: 0; margin-bottom: 0.3rem; font-weight: 600; opacity: 0.75; }

  @media (max-width: 576px) {
    #octavia-manager .om-tree-lvl1 { margin-left: 0.75rem; }
    #octavia-manager .om-tree-lvl2 { margin-left: 1.5rem; }
    #octavia-manager .om-tree-lvl3 { margin-left: 2.25rem; }
    #octavia-manager .om-cmd-label { min-width: 68px; font-size: 0.72rem; }
  }
</style>

<div id="octavia-manager">

  <div class="om-tabs">
    <button class="om-tab-btn active" data-panel="structure">리소스 구조</button>
    <button class="om-tab-btn" data-panel="cli">CLI 명령어 / 로그</button>
    <button class="om-tab-btn" data-panel="cleanup">안 지워지는 LB 삭제</button>
    <button class="om-tab-btn" data-panel="service">서비스 상태/재시작</button>
    <button class="om-tab-btn" data-panel="troubleshoot">트러블슈팅</button>
  </div>

  <!-- 구조 -->
  <div class="om-panel active" id="panel-structure">
    <div class="om-card">
      <h3><i class="fas fa-sitemap"></i> 리소스 계층 구조</h3>
      <img src="/assets/img/posts/octavia-loadbalancer-structure.png" alt="LoadBalancer 리소스 계층 구조도" style="max-width:100%; height:auto; display:block; margin:0.5rem auto 1.25rem; border-radius:8px;">
      <p>위에서 아래로 갈수록 하위 리소스입니다. 색상으로 계층 깊이를 구분했습니다.</p>

      <div class="om-tree">
        <div class="om-tree-node om-tree-lvl0">
          <div class="om-tree-icon"><i class="fas fa-server"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">LoadBalancer</div>
            <div class="om-tree-desc">최상위 객체. VIP(가상 IP)를 가지며 실제로는 <strong>Amphora</strong>(Nova VM) 위에서 동작합니다. 생성 시 Amphora가 배포되고, 하위 리소스(Listener, Pool 등)가 여기 종속됩니다.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl1">
          <div class="om-tree-icon"><i class="fas fa-plug"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">Listener <span class="om-tree-tag">HTTP · HTTPS · TCP · TERMINATED_HTTPS</span></div>
            <div class="om-tree-desc">트래픽을 수신하는 지점. 프로토콜/포트를 정의합니다. 하나의 LB에 여러 Listener를 둘 수 있습니다(예: 80, 443).</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl2">
          <div class="om-tree-icon"><i class="fas fa-layer-group"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">Pool <span class="om-tree-tag">default_pool</span></div>
            <div class="om-tree-desc">실제 트래픽을 분산시킬 백엔드 서버 그룹. 로드밸런싱 알고리즘(ROUND_ROBIN, LEAST_CONNECTIONS, SOURCE_IP 등)을 지정합니다.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl3">
          <div class="om-tree-icon"><i class="fas fa-desktop"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">Pool Member <span class="om-tree-tag">× N</span></div>
            <div class="om-tree-desc">Pool에 속한 개별 백엔드 서버. IP:Port와 가중치(weight)를 가지며, Health Monitor 결과에 따라 개별적으로 활성/비활성 상태가 바뀝니다.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl3">
          <div class="om-tree-icon"><i class="fas fa-heartbeat"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">Health Monitor</div>
            <div class="om-tree-desc">Pool Member 상태를 주기적으로 점검(HTTP/TCP/PING). 연속 실패 횟수(<code>max_retries_down</code>)를 넘기면 <code>OFFLINE</code> 처리합니다.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl2">
          <div class="om-tree-icon"><i class="fas fa-code-branch"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">L7 Policy <span class="om-tree-tag">선택 · × N</span></div>
            <div class="om-tree-desc">L7 Rule(path/header/cookie) 조건에 따라 트래픽을 다른 Pool로 redirect합니다.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl1">
          <div class="om-tree-icon"><i class="fas fa-globe"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">VIP</div>
            <div class="om-tree-desc">Virtual IP — 실제 트래픽이 들어오는 주소.</div>
          </div>
        </div>

        <div class="om-tree-node om-tree-lvl1">
          <div class="om-tree-icon"><i class="fas fa-sync-alt"></i></div>
          <div class="om-tree-body">
            <div class="om-tree-title">VRRP Group <span class="om-tree-tag">HA 구성 시</span></div>
            <div class="om-tree-desc">Amphora 이중화(Active/Standby)용 메타데이터.</div>
          </div>
        </div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-shield-alt"></i> Amphora Active/Standby (HA) 동작 흐름</h3>
      <p>HA로 구성하면 Amphora가 2대(Active/Standby) 뜨고, VRRP(keepalived)로 서로의 상태를 감시합니다.</p>
      <pre class="om-ha-diagram"><code>[평상시]
  Amphora A (MASTER)  &#8592; VIP 트래픽 처리, VRRP 광고 주기적 전송
  Amphora B (BACKUP)  &#8592; 대기 상태, A의 VRRP 광고 수신 대기

[Amphora A 장애 발생]
  A 응답 없음
    &#8594; octavia-health-manager가 감지
    &#8594; A의 VRRP 광고 중단
    &#8594; B가 타임아웃 감지 &#8594; 스스로 MASTER로 승격 (VIP 인수)
    &#8594; health-manager가 A를 재생성(rebuild) 트리거 (필요 시)</code></pre>
      <p style="margin-top:0.75rem">그래서 장애 시 VIP 자체는 끊기지 않고 유지되며, 문제가 있던 Amphora만 백그라운드에서 교체됩니다. 수동으로 즉시 전환하고 싶다면 "CLI / 로그" 탭의 <code>amphora failover</code> 명령을 사용합니다.</p>
    </div>
  </div>

  <!-- CLI / 로그 -->
  <div class="om-panel" id="panel-cli">
    <div class="om-card">
      <h3><i class="fas fa-terminal"></i> OpenStack CLI 사용 방법</h3>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">LB 목록 조회</span>
          <code>openstack loadbalancer list</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
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
        <tr><th>리소스</th><th>목록</th><th>상세</th></tr>
        <tr><td>LoadBalancer</td>
          <td><code>openstack loadbalancer list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer show &lt;lb-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>Listener</td>
          <td><code>openstack loadbalancer listener list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer listener show &lt;listener-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>Pool</td>
          <td><code>openstack loadbalancer pool list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer pool show &lt;pool-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>Pool Member</td>
          <td><code>openstack loadbalancer member list &lt;pool-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer member show &lt;pool-id&gt; &lt;member-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>Health Monitor</td>
          <td><code>openstack loadbalancer healthmonitor list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer healthmonitor show &lt;hm-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>L7 Policy</td>
          <td><code>openstack loadbalancer l7policy list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer l7policy show &lt;policy-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>L7 Rule</td>
          <td><code>openstack loadbalancer l7rule list &lt;policy-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer l7rule show &lt;policy-id&gt; &lt;rule-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
        <tr><td>Amphora</td>
          <td><code>openstack loadbalancer amphora list</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
          <td><code>openstack loadbalancer amphora show &lt;amphora-id&gt;</code><button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button></td>
        </tr>
      </table>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-tools"></i> 운영에 자주 쓰는 명령어</h3>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">트래픽/통계</span>
          <code>openstack loadbalancer stats show &lt;lb-id&gt;</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">강제 failover</span>
          <code>openstack loadbalancer amphora failover &lt;amphora-id&gt;</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">강제 삭제</span>
          <code>openstack loadbalancer delete --cascade &lt;lb-id&gt;</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">쿼터 확인</span>
          <code>openstack loadbalancer quota show &lt;project-id&gt;</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-file-alt"></i> Octavia 로그 확인</h3>
      <p>Octavia 각 서비스의 로그는 <code>/var/log/octavia/</code>에 서비스별로 분리되어 쌓입니다 (octavia-api.log, octavia-worker.log, octavia-health-manager.log, octavia-housekeeping.log).</p>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">디렉터리 이동</span>
          <code>cd /var/log/octavia &amp;&amp; ls -la</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">API 로그</span>
          <code>tail -f octavia-api.log</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">Worker 로그</span>
          <code>tail -f octavia-worker.log</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
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
      <p>DB를 직접 삭제하기 전에, 이 LB에 딸린 리소스가 실제로 몇 개나 있는지 하나씩 조회해서 확인합니다. 각 쿼리는 개별적으로 복사할 수 있습니다.</p>
      <span class="om-generated-badge" id="om-select-badge">위 입력창의 ID로 자동 생성됨 (미입력 시 예시 ID 사용)</span>
      <div class="om-cmd-list" id="om-select-list"></div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-database"></i> ④ 그래도 안 지워지면: DB 직접 삭제</h3>
      <div class="om-note">
        최후 수단입니다. 아래 순서(1→10)를 반드시 지켜야 참조 무결성 오류가 나지 않습니다. 한 단계씩 복사해서 실행하며 결과를 확인하는 걸 권장합니다. 실행 전 반드시 백업하고, 위 ③ 조회 결과로 대상이 맞는지 먼저 확인하세요.
      </div>
      <span class="om-generated-badge" id="om-sql-badge">위 입력창의 ID로 자동 생성됨 (미입력 시 예시 ID 사용)</span>
      <div class="om-cmd-list" id="om-delete-list"></div>
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
      <div class="om-cmd-list" style="margin-top:0.75rem">
        <div class="om-cmd-row">
          <span class="om-cmd-label">전체 목록</span>
          <code>systemctl list-units | grep octavia</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-terminal"></i> 상태 확인</h3>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">api</span>
          <code>systemctl status octavia-api</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">worker</span>
          <code>systemctl status octavia-worker</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">health-manager</span>
          <code>systemctl status octavia-health-manager</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">housekeeping</span>
          <code>systemctl status octavia-housekeeping</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">interface</span>
          <code>systemctl status octavia-interface</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-redo"></i> 재시작</h3>
      <p>재시작 후에는 위 "상태 확인" 명령어로 정상 기동됐는지 다시 확인하세요.</p>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">worker</span>
          <code>systemctl restart octavia-worker</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">health-manager</span>
          <code>systemctl restart octavia-health-manager</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">housekeeping</span>
          <code>systemctl restart octavia-housekeeping</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="om-cmd-row">
          <span class="om-cmd-label">api</span>
          <code>systemctl restart octavia-api</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-network-wired"></i> o-hm0 인터페이스 확인</h3>
      <p>Octavia가 Amphora와 통신하는 데 쓰는 헬스매니저 전용 인터페이스입니다. 이게 없으면 LB 생성 자체가 실패합니다.</p>
      <div class="om-cmd-list">
        <div class="om-cmd-row">
          <span class="om-cmd-label">인터페이스 확인</span>
          <code>ip a | grep o-hm0</code>
          <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
    </div>
  </div>

  <!-- 트러블슈팅 -->
  <div class="om-panel" id="panel-troubleshoot">
    <div class="om-card">
      <h3><i class="fas fa-stethoscope"></i> 기본 진단 흐름</h3>

      <div class="om-step">
        <div class="om-step-num">1</div>
        <div class="om-step-body">
          LB 상태 확인 — <code>provisioning_status</code>, <code>operating_status</code> 확인
          <div class="om-cmd-list">
            <div class="om-cmd-row">
              <code>openstack loadbalancer show &lt;lb-id&gt;</code>
              <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
            </div>
          </div>
        </div>
      </div>
      <div class="om-step">
        <div class="om-step-num">2</div>
        <div class="om-step-body">
          Amphora 상태 확인
          <div class="om-cmd-list">
            <div class="om-cmd-row">
              <code>openstack loadbalancer amphora list --loadbalancer &lt;lb-id&gt;</code>
              <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
            </div>
          </div>
        </div>
      </div>
      <div class="om-step">
        <div class="om-step-num">3</div>
        <div class="om-step-body">
          서비스 로그 확인
          <div class="om-cmd-list">
            <div class="om-cmd-row">
              <code>tail -f /var/log/octavia/octavia-worker.log</code>
              <button class="om-cmd-copy" onclick="omCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
            </div>
          </div>
        </div>
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

function omCopyInline(btn) {
  var code = btn.previousElementSibling;
  if (!code) return;
  navigator.clipboard.writeText(code.innerText).then(function () {
    var orig = btn.innerHTML;
    btn.innerHTML = '<i class="fas fa-check"></i>';
    btn.classList.add('copied');
    setTimeout(function () {
      btn.innerHTML = orig;
      btn.classList.remove('copied');
    }, 1000);
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

  omRenderSteps('om-select-list', [
    { label: 'LoadBalancer', sql: "SELECT * FROM load_balancer WHERE id = '" + id + "';" },
    { label: 'Listener', sql: "SELECT * FROM listener WHERE load_balancer_id = '" + id + "';" },
    { label: 'Pool', sql: "SELECT * FROM pool WHERE load_balancer_id = '" + id + "';" },
    { label: 'Pool Member', sql: "SELECT * FROM member\nWHERE pool_id IN (SELECT id FROM pool WHERE load_balancer_id = '" + id + "');" },
    { label: 'Health Monitor', sql: "SELECT * FROM health_monitor\nWHERE pool_id IN (SELECT id FROM pool WHERE load_balancer_id = '" + id + "');" },
    { label: 'Amphora', sql: "SELECT * FROM amphora WHERE load_balancer_id = '" + id + "';" },
    { label: 'VIP', sql: "SELECT * FROM vip WHERE load_balancer_id = '" + id + "';" },
    { label: 'VRRP Group', sql: "SELECT * FROM vrrp_group WHERE load_balancer_id = '" + id + "';" }
  ]);

  var cascadeEl = document.getElementById('om-cascade-cmd');
  if (cascadeEl) {
    cascadeEl.textContent = 'openstack loadbalancer delete --cascade ' + id;
  }

  omRenderSteps('om-delete-list', [
    { label: 'Health Monitor 삭제', sql: "DELETE FROM health_monitor\nWHERE pool_id IN (\n    SELECT id FROM pool\n    WHERE load_balancer_id = '" + id + "'\n);" },
    { label: 'Pool Member 삭제', sql: "DELETE FROM member\nWHERE pool_id IN (\n    SELECT id FROM pool\n    WHERE load_balancer_id = '" + id + "'\n);" },
    { label: 'Listener → default_pool_id 참조 끊기', sql: "UPDATE listener\nSET default_pool_id = NULL\nWHERE default_pool_id IN (\n    SELECT id FROM pool\n    WHERE load_balancer_id = '" + id + "'\n);" },
    { label: 'Pool 삭제', sql: "DELETE FROM pool\nWHERE load_balancer_id = '" + id + "';" },
    { label: 'Listener 삭제', sql: "DELETE FROM listener\nWHERE load_balancer_id = '" + id + "';" },
    { label: 'amphora_health 삭제', sql: "DELETE FROM amphora_health\nWHERE amphora_id IN (\n    SELECT id FROM amphora\n    WHERE load_balancer_id = '" + id + "'\n);" },
    { label: 'amphora 삭제', sql: "DELETE FROM amphora\nWHERE load_balancer_id = '" + id + "';" },
    { label: 'VIP 삭제', sql: "DELETE FROM vip\nWHERE load_balancer_id = '" + id + "';" },
    { label: 'vrrp_group 삭제 (Amphora HA 메타 테이블)', sql: "DELETE FROM vrrp_group\nWHERE load_balancer_id = '" + id + "';" },
    { label: 'LoadBalancer 삭제', sql: "DELETE FROM load_balancer\nWHERE id = '" + id + "';" }
  ]);
}

function omRenderSteps(containerId, steps) {
  var container = document.getElementById(containerId);
  if (!container) return;
  container.innerHTML = '';
  steps.forEach(function (step, i) {
    var wrap = document.createElement('div');
    wrap.className = 'om-sql-step';

    var label = document.createElement('div');
    label.className = 'om-cmd-label';
    label.textContent = (i + 1) + '. ' + step.label;

    var copyWrap = document.createElement('div');
    copyWrap.className = 'om-copy-wrap';

    var btn = document.createElement('button');
    btn.className = 'om-copy-btn';
    btn.textContent = '복사';
    btn.onclick = function () { omCopy(btn); };

    var pre = document.createElement('pre');
    var code = document.createElement('code');
    code.textContent = step.sql;
    pre.appendChild(code);

    copyWrap.appendChild(btn);
    copyWrap.appendChild(pre);
    wrap.appendChild(label);
    wrap.appendChild(copyWrap);
    container.appendChild(wrap);
  });
}
</script>
