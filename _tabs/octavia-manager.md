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
  #octavia-manager .om-header {
    padding-bottom: 1rem;
    margin-bottom: 1.5rem;
    border-bottom: 2px solid var(--link-color);
  }
  #octavia-manager .om-header h1 {
    font-size: 1.6rem;
    font-weight: 700;
    letter-spacing: -0.02em;
    margin-bottom: 0.4rem;
  }
  #octavia-manager .om-header p {
    font-size: 0.95rem;
    opacity: 0.65;
    margin: 0;
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

  #octavia-manager .om-diagram {
    font-family: var(--font-monospace, monospace);
    font-size: 0.85rem;
    line-height: 1.9;
    white-space: pre;
    overflow-x: auto;
    background: var(--code-bg, #f6f6f6);
    border-radius: 8px;
    padding: 1rem 1.2rem;
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
</style>

<div id="octavia-manager">

  <div class="om-header">
    <h1><i class="fas fa-random"></i> Octavia Manager</h1>
    <p>Octavia 로드밸런서의 구조, 운영, 트러블슈팅을 정리한 레퍼런스입니다. 실제 API를 호출하지 않는 정적 참고 자료입니다.</p>
  </div>

  <div class="om-tabs">
    <button class="om-tab-btn active" data-panel="structure">구조</button>
    <button class="om-tab-btn" data-panel="cleanup">정리 SQL</button>
    <button class="om-tab-btn" data-panel="service">서비스 관리</button>
    <button class="om-tab-btn" data-panel="troubleshoot">트러블슈팅</button>
  </div>

  <!-- 구조 -->
  <div class="om-panel active" id="panel-structure">
    <div class="om-card">
      <h3><i class="fas fa-sitemap"></i> 리소스 계층 구조</h3>
      <div class="om-diagram">LoadBalancer
  └── Listener            (프로토콜/포트: HTTP, HTTPS, TCP, TERMINATED_HTTPS ...)
        ├── Pool (default_pool)
        │     ├── Pool Member × N   (백엔드 서버 IP:Port, weight)
        │     └── Health Monitor    (delay, timeout, max_retries, http_method ...)
        └── L7 Policy × N (선택)
              ├── L7 Rule × N       (조건: path, header, cookie ...)
              └── redirect → 다른 Pool</div>
      <p style="margin-top:0.9rem">하나의 LoadBalancer는 여러 Listener를 가질 수 있고, 각 Listener는 기본 Pool(default_pool) 하나와 선택적으로 여러 L7 Policy를 가집니다. L7 Policy는 조건(L7 Rule)에 따라 트래픽을 다른 Pool로 보낼 때 사용합니다.</p>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-heartbeat"></i> Health Monitor의 역할</h3>
      <p>Pool Member의 상태를 주기적으로 점검해서 비정상 서버를 트래픽 분배 대상에서 자동으로 제외합니다. HTTP/TCP/PING 등 방식으로 체크하며, 연속 실패 횟수(<code>max_retries_down</code>)를 넘기면 <code>OFFLINE</code>으로 표시됩니다.</p>
    </div>
  </div>

  <!-- 정리 SQL -->
  <div class="om-panel" id="panel-cleanup">
    <div class="om-note">
      API로 정상 삭제되지 않고 <code>PENDING_DELETE</code>/<code>ERROR</code> 상태에 멈춘 LB를 정리할 때만 최후 수단으로 사용합니다. 운영 DB 직접 수정은 리스크가 크므로, 반드시 백업 후 트랜잭션 내에서 실행하고 가능하면 API 삭제를 먼저 시도하세요.
    </div>
    <div class="om-card">
      <h3><i class="fas fa-trash-alt"></i> 삭제 순서가 중요한 이유</h3>
      <p>하위 리소스가 상위 리소스를 참조(FK)하고 있어서, 자식 → 부모 순서로 지워야 참조 무결성 오류가 나지 않습니다. (Health Monitor/Member → Pool, L7 Policy → Listener, Listener/Pool → LoadBalancer)</p>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>-- 1. Pool의 자식부터 삭제
DELETE FROM health_monitor WHERE pool_id IN ('pool-uuid-1');
DELETE FROM member WHERE pool_id IN ('pool-uuid-1');

-- 2. Pool 삭제
DELETE FROM pool WHERE load_balancer_id = 'lb-uuid';

-- 3. Listener의 자식(L7 Policy) 삭제
DELETE FROM l7policy WHERE listener_id IN ('listener-uuid-1');

-- 4. Listener 삭제
DELETE FROM listener WHERE load_balancer_id = 'lb-uuid';

-- 5. 마지막으로 LoadBalancer 삭제
DELETE FROM load_balancer WHERE id = 'lb-uuid';</code></pre>
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
      </table>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-terminal"></i> 상태 확인</h3>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>systemctl status octavia-api
systemctl status octavia-worker
systemctl status octavia-health-manager
systemctl status octavia-housekeeping</code></pre>
      </div>
    </div>

    <div class="om-card">
      <h3><i class="fas fa-redo"></i> 재시작</h3>
      <div class="om-copy-wrap">
        <button class="om-copy-btn" onclick="omCopy(this)">복사</button>
        <pre><code>systemctl restart octavia-api
systemctl restart octavia-worker
systemctl restart octavia-health-manager
systemctl restart octavia-housekeeping</code></pre>
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
        <div class="om-step-body">서비스 로그 확인<br><code>journalctl -u octavia-worker -f</code>, <code>journalctl -u octavia-health-manager -f</code></div>
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
          <td><code>PENDING_DELETE</code>에서 멈춘 LB</td>
          <td>API 재삭제 재시도 → 계속 실패 시 "정리 SQL" 탭 참고</td>
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
</script>
