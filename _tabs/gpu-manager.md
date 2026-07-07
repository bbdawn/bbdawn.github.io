---
layout: page
title: GPU Manager
icon: fas fa-toolbox
order: 4
permalink: /gpu-manager/
---

<style>
  #gpu-manager * { box-sizing: border-box; }
  #gpu-manager { margin: -0.5rem 0 0; }
  #gpu-manager .gm-tabs {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
  }
  #gpu-manager .gm-tab-btn {
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
  #gpu-manager .gm-tab-btn:hover { border-color: var(--link-color); }
  #gpu-manager .gm-tab-btn.active {
    background: var(--link-color);
    border-color: var(--link-color);
    color: #fff;
  }

  #gpu-manager .gm-panel { display: none; }
  #gpu-manager .gm-panel.active { display: block; }

  #gpu-manager .gm-card {
    border: 1px solid var(--border-color, #dee2e6);
    border-radius: 10px;
    padding: 1.25rem 1.4rem;
    margin-bottom: 1.25rem;
    background: var(--card-bg, rgba(0,0,0,0.015));
  }
  #gpu-manager .gm-card h3 {
    font-size: 1.05rem;
    font-weight: 700;
    margin-bottom: 0.6rem;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  #gpu-manager .gm-card h3 i { color: var(--link-color); opacity: 0.85; }
  #gpu-manager .gm-card p { font-size: 0.92rem; line-height: 1.75; opacity: 0.85; }

  #gpu-manager .gm-note {
    font-size: 0.85rem;
    line-height: 1.8;
    padding: 0.7rem 1rem;
    border-left: 3px solid var(--link-color);
    background: rgba(var(--bs-primary-rgb, 13,110,253), 0.06);
    border-radius: 4px;
    margin: 0.75rem 0;
  }
  #gpu-manager .gm-note.warn {
    border-left-color: #dc3545;
    background: rgba(220,53,69,0.06);
  }

  #gpu-manager pre { overflow-x: auto; font-size: 0.82rem; }

  #gpu-manager table.gm-table {
    width: 100%;
    font-size: 0.88rem;
    border-collapse: collapse;
    margin: 0.75rem 0 0.25rem;
  }
  #gpu-manager table.gm-table th,
  #gpu-manager table.gm-table td {
    border: 1px solid var(--border-color, #dee2e6);
    padding: 0.5rem 0.7rem;
    text-align: left;
    vertical-align: top;
  }
  #gpu-manager table.gm-table th { background: rgba(0,0,0,0.03); }

  #gpu-manager .gm-cmd-list { display: flex; flex-direction: column; gap: 0.4rem; margin: 0.75rem 0 0.25rem; }
  #gpu-manager .gm-cmd-row {
    display: flex; align-items: center; gap: 0.6rem;
    background: var(--code-bg, #f6f6f6);
    border-radius: 6px;
    padding: 0.45rem 0.5rem 0.45rem 0.75rem;
  }
  #gpu-manager .gm-cmd-label { flex-shrink: 0; min-width: 130px; font-size: 0.78rem; opacity: 0.6; }
  #gpu-manager .gm-cmd-row code { flex: 1; min-width: 0; background: none; padding: 0; font-size: 0.82rem; white-space: pre; overflow-x: auto; }
  #gpu-manager .gm-cmd-copy {
    flex-shrink: 0; border: none; background: none; cursor: pointer;
    color: var(--text-muted-color, #6c757d);
    padding: 0.25rem 0.45rem; border-radius: 5px;
    font-size: 0.85rem; transition: all 0.15s;
  }
  #gpu-manager .gm-cmd-copy:hover { color: var(--link-color); background: rgba(0,0,0,0.06); }
  #gpu-manager .gm-cmd-copy.copied { color: #28a745; }

  #gpu-manager .gm-github-btn {
    display: inline-flex; align-items: center; gap: 0.5rem;
    background: #24292f; color: #fff !important;
    border-radius: 8px; padding: 0.6rem 1.1rem;
    font-size: 0.92rem; font-weight: 600;
    text-decoration: none !important;
    margin-bottom: 0.5rem;
  }
  #gpu-manager .gm-github-btn:hover { background: #000; }
  #gpu-manager .gm-github-btn i { font-size: 1.1rem; }

  #gpu-manager .gm-feature-shot {
    border: 1px solid var(--border-color, #dee2e6);
    border-radius: 8px;
    padding: 0.75rem;
    margin: 0.75rem 0 1rem;
    background: var(--code-bg, #f6f6f6);
    text-align: center;
    font-size: 0.85rem;
    opacity: 0.7;
  }
  #gpu-manager .gm-feature-shot img { max-width: 100%; height: auto; border-radius: 6px; display: block; margin: 0 auto; }

  #gpu-manager .gm-compare-badge {
    display: inline-block; font-size: 0.72rem; font-weight: 700;
    border-radius: 4px; padding: 0.1rem 0.5rem; margin-right: 0.4rem;
  }
  #gpu-manager .gm-compare-badge.before { background: rgba(220,53,69,0.12); color: #dc3545; }
  #gpu-manager .gm-compare-badge.after { background: rgba(40,167,69,0.12); color: #28a745; }

  @media (max-width: 576px) {
    #gpu-manager .gm-cmd-label { min-width: 90px; font-size: 0.72rem; }
  }
</style>

<div id="gpu-manager">

  <div class="gm-tabs">
    <button class="gm-tab-btn active" data-panel="concept">GPU 가상화 개념 (IaaS 관점)</button>
    <button class="gm-tab-btn" data-panel="tool">gpu-manager 소개</button>
    <button class="gm-tab-btn" data-panel="before">이전 방식 (수동 스크립트)</button>
  </div>

  <!-- ══════════ GPU 가상화 개념 ══════════ -->
  <div class="gm-panel active" id="panel-concept">
    <div class="gm-card">
      <h3><i class="fas fa-layer-group"></i> IaaS에서 GPU를 인스턴스에 제공하는 3가지 방식</h3>
      <p>OpenStack 같은 IaaS 플랫폼에서 물리 GPU를 가상 인스턴스에 할당하는 방법은 격리 수준과 자원 활용률이 서로 트레이드오프 관계에 있는 세 가지로 나뉩니다.</p>
      <table class="gm-table">
        <tr><th>방식</th><th>격리 단위</th><th>GPU 활용률</th><th>대표 사용 사례</th></tr>
        <tr>
          <td><strong>GPU Passthrough</strong></td>
          <td>물리 GPU 전체를 VM 1개에 독점 할당</td>
          <td>가장 낮음 (VM 1개가 GPU 전체 점유)</td>
          <td>대규모 학습, GPU 성능을 100% 그대로 써야 하는 워크로드</td>
        </tr>
        <tr>
          <td><strong>vGPU</strong></td>
          <td>NVIDIA GRID 드라이버가 GPU를 소프트웨어적으로 시분할</td>
          <td>중간 (여러 VM이 시간 단위로 공유)</td>
          <td>VDI, 다수의 경량 그래픽/추론 워크로드</td>
        </tr>
        <tr>
          <td><strong>MIG (Multi-Instance GPU)</strong></td>
          <td>Ampere 이상 GPU의 하드웨어 파티션(SM/메모리 물리 분리)</td>
          <td>높음 (파티션 단위로 여러 VM이 동시에 안전하게 공유)</td>
          <td>추론 서빙, 소규모 학습 다중 테넌시</td>
        </tr>
      </table>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-microchip"></i> GPU Passthrough</h3>
      <p>호스트의 물리 GPU를 PCI Passthrough로 VM에 그대로 넘겨주는 방식입니다. Nova 기준으로는 <code>nova.conf</code>의 PCI alias와 Flavor의 <code>extra_specs</code>(<code>pci_passthrough:alias</code>)를 매칭시켜 스케줄링합니다.</p>
      <ul style="font-size:0.92rem; line-height:1.8; opacity:0.85; padding-left:1.2rem;">
        <li>장점: 드라이버 오버헤드가 거의 없어 네이티브에 가까운 성능</li>
        <li>단점: GPU 1개 = VM 1개로 고정되어 자원 활용률이 낮고, 유연한 재배치가 어려움</li>
      </ul>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-clone"></i> vGPU</h3>
      <p>NVIDIA vGPU(GRID) 드라이버가 하이퍼바이저 레벨에서 물리 GPU를 여러 개의 가상 GPU 프로파일로 나누고, 각 VM은 시분할 스케줄링으로 GPU 연산 자원을 순환 사용합니다.</p>
      <ul style="font-size:0.92rem; line-height:1.8; opacity:0.85; padding-left:1.2rem;">
        <li>장점: 하나의 물리 GPU를 여러 VM이 동시에 쓸 수 있어 활용률이 높음</li>
        <li>단점: 시분할 특성상 워크로드가 몰리면 VM 간 성능 간섭이 발생할 수 있음, 별도 라이선스 필요</li>
      </ul>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-th"></i> MIG (Multi-Instance GPU)</h3>
      <p>NVIDIA Ampere 이상 아키텍처(A100, H100 등)에서 지원하는 하드웨어 레벨 파티셔닝입니다. SM(Streaming Multiprocessor)과 메모리를 물리적으로 분리한 <strong>GPU Instance</strong> 단위로 나누고, 그 안에 <strong>Compute Instance</strong>를 만들어 각 VM/컨테이너에 독립적인 자원 슬라이스를 할당합니다. OpenStack에서는 각 MIG 슬라이스가 하나의 <code>mdev</code>(mediated device)로 노출되어 VM에 할당됩니다.</p>
      <ul style="font-size:0.92rem; line-height:1.8; opacity:0.85; padding-left:1.2rem;">
        <li>장점: vGPU와 달리 하드웨어 레벨에서 격리되어 있어 VM 간 성능 간섭이 없음(진짜 물리적 파티션)</li>
        <li>단점: 지원 GPU 모델이 제한적이고, 프로파일(1g.5gb, 2g.10gb 등) 조합에 제약이 있어 운영 설계가 까다로움</li>
      </ul>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-balance-scale"></i> IaaS 운영자 관점의 선택 기준</h3>
      <p>결국 <strong>격리 수준(보안/성능 예측 가능성)</strong>과 <strong>GPU 활용률(비용 효율)</strong> 사이에서 워크로드 성격에 맞는 지점을 고르는 문제입니다. 대규모 학습처럼 GPU를 통째로 오래 점유하는 워크로드는 Passthrough가, 다수의 경량 추론/개발 워크로드를 태워야 하는 멀티테넌시 환경은 MIG/vGPU가 유리합니다. 실제 운영에서는 같은 물리 GPU 풀 안에 Passthrough용과 MIG용을 나눠 구성하고, 수요에 따라 재구성하는 경우가 많습니다.</p>
    </div>
  </div>

  <!-- ══════════ gpu-manager 소개 ══════════ -->
  <div class="gm-panel" id="panel-tool">
    <div class="gm-note warn">
      <strong>TODO</strong>: 아래 GitHub 링크는 placeholder입니다. 실제 저장소 URL로 교체해주세요.
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-lightbulb"></i> 왜 만들었는가</h3>
      <p>GPU를 MIG로 운영하다 보면 mdev 자원이 orphan 상태로 남는 경우가 있는데, 이를 판단하려면 여러 명령어를 조합해 직접 확인해야 했고 판단까지 약 10분이 걸렸습니다. 이 지식이 특정인(본인)에게만 있다 보니, 도메인을 모르는 QA에게 판단 기준을 설명하는 데도 오랜 시간이 걸렸고 관련 문의가 항상 본인에게 돌아왔습니다. GPU Passthrough/MIG 할당 현황을 매번 여러 명령어로 조회하고, 문제 있는 리소스를 수동으로 정리하는 반복 작업 자체를 도구로 없애기 위해 gpu-manager를 만들었습니다.</p>
    </div>

    <div class="gm-card">
      <h3><i class="fab fa-github"></i> 저장소</h3>
      <a class="gm-github-btn" href="#" target="_blank" rel="noopener">
        <i class="fab fa-github"></i> GitHub에서 gpu-manager 보기
      </a>
      <p>CLI/TUI 형태로 동작하며, GPU Passthrough·MIG 인스턴스 현황 조회, mdev orphan 탐지 및 정리, MIG 프로파일 생성/삭제를 UI 조작만으로 처리할 수 있게 만든 도구입니다.</p>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-images"></i> 주요 기능</h3>

      <p><strong>1. GPU 현황 대시보드</strong></p>
      <div class="gm-feature-shot">
        (스크린샷 자리 — GPU Host/Instance 전체 현황 화면)
      </div>
      <p>호스트별 GPU 목록, 각 GPU의 Passthrough/MIG 사용 여부, 할당된 인스턴스를 한 화면에서 조회합니다.</p>

      <p><strong>2. mdev orphan 탐지 및 정리</strong></p>
      <div class="gm-feature-shot">
        (스크린샷 자리 — orphan mdev 탐지 목록 및 삭제 화면)
      </div>
      <p>더 이상 어떤 인스턴스에도 연결되지 않은 mdev 디바이스를 자동으로 판별해 목록으로 보여주고, 클릭 한 번으로 삭제까지 실행합니다.</p>

      <p><strong>3. MIG 프로파일 관리</strong></p>
      <div class="gm-feature-shot">
        (스크린샷 자리 — MIG 프로파일 생성/삭제 화면)
      </div>
      <p>GPU별로 어떤 MIG 프로파일이 몇 개 만들어져 있는지 보여주고, 새 GPU Instance/Compute Instance 생성이나 삭제를 명령어 없이 처리합니다.</p>

      <p style="margin-top:1rem; font-size:0.85rem; opacity:0.6;">※ 위 스크린샷 3장은 자리만 잡아둔 상태입니다. reorder 도구의 글 편집 모드로 이미지를 업로드하신 뒤, 실제 경로로 교체해주세요.</p>
    </div>
  </div>

  <!-- ══════════ 이전 방식 (수동 스크립트) ══════════ -->
  <div class="gm-panel" id="panel-before">
    <div class="gm-note">
      gpu-manager가 없던 시절, GPU Passthrough/MIG 상태를 확인하고 정리하려면 아래처럼 여러 명령어를 순서대로 조합해야 했습니다.
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-search"></i> mdev orphan 판단 — 수동 명령어 조합</h3>
      <p><span class="gm-compare-badge before">BEFORE</span>여러 명령어를 조합해 mdev가 실제로 어떤 인스턴스에 연결되어 있는지 하나씩 대조해야 했고, 본인 기준으로도 판단까지 약 10분이 걸렸습니다.</p>
      <div class="gm-cmd-list">
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">현재 mdev 목록</span>
          <code>ls /sys/class/mdev_bus/*/mdev_supported_types/*/devices/</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">GPU MIG 인스턴스 목록</span>
          <code>nvidia-smi mig -lgi</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">Nova가 인식 중인 VM-mdev 매핑</span>
          <code>virsh dumpxml &lt;instance-id&gt; | grep mdev</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">DB상 인스턴스 존재 여부 확인</span>
          <code>openstack server show &lt;instance-id&gt;</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
      <p style="margin-top:0.75rem; font-size:0.85rem; opacity:0.6;">※ 위 명령어는 대표적인 조합을 재구성한 예시입니다. 실제로 조합하신 정확한 명령어로 교체해주세요.</p>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-th-large"></i> MIG 프로파일 관리 — 수동 명령어 조합</h3>
      <p><span class="gm-compare-badge before">BEFORE</span>MIG 프로파일을 만들고 지울 때도 아래 명령어들을 GPU마다, 프로파일마다 반복 실행해야 했습니다.</p>
      <div class="gm-cmd-list">
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">MIG 모드 활성화</span>
          <code>nvidia-smi -i &lt;gpu-id&gt; -mig 1</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">지원 프로파일 확인</span>
          <code>nvidia-smi mig -lgip</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">GPU Instance 생성</span>
          <code>nvidia-smi mig -cgi &lt;profile-id&gt; -C</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">생성된 GPU/Compute Instance 확인</span>
          <code>nvidia-smi mig -lgi &amp;&amp; nvidia-smi mig -lci</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">Compute Instance 삭제</span>
          <code>nvidia-smi mig -dci -ci &lt;ci-id&gt; -gi &lt;gi-id&gt;</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
        <div class="gm-cmd-row">
          <span class="gm-cmd-label">GPU Instance 삭제</span>
          <code>nvidia-smi mig -dgi -gi &lt;gi-id&gt;</code>
          <button class="gm-cmd-copy" onclick="gmCopyInline(this)" title="복사"><i class="fas fa-copy"></i></button>
        </div>
      </div>
      <p style="margin-top:0.75rem; font-size:0.85rem; opacity:0.6;">※ 이 부분도 실제로 사용하신 정확한 명령어/순서로 교체해주세요 (표준 nvidia-smi mig 명령어를 기준으로 재구성했습니다).</p>
    </div>

    <div class="gm-card">
      <h3><i class="fas fa-arrow-right"></i> gpu-manager로 바뀐 것</h3>
      <p><span class="gm-compare-badge after">AFTER</span>위 명령어들을 순서와 문법까지 정확히 기억해야 했던 작업이, gpu-manager에서는 클릭 몇 번으로 현황 파악부터 정리까지 즉시 처리됩니다. 전문 지식이 없는 인력도 도구를 통해 바로 운영에 참여할 수 있게 되었습니다.</p>
    </div>
  </div>
</div>

<script>
(function () {
  document.querySelectorAll('#gpu-manager .gm-tab-btn').forEach(function (btn) {
    btn.addEventListener('click', function () {
      document.querySelectorAll('#gpu-manager .gm-tab-btn').forEach(function (b) { b.classList.remove('active'); });
      document.querySelectorAll('#gpu-manager .gm-panel').forEach(function (p) { p.classList.remove('active'); });
      btn.classList.add('active');
      document.getElementById('panel-' + btn.dataset.panel).classList.add('active');
    });
  });
})();

function gmCopyInline(btn) {
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
</script>
