---
icon: fas fa-info-circle
order: 0
---

# 이주현

<p style="font-size: 1.1rem; font-weight: 500; line-height: 1.7; margin: 0.25rem 0 0.75rem;">
  개발도 하고 인프라도 아는 클라우드 백엔드 개발자입니다.
</p>
<p style="font-size: 1rem; line-height: 1.85; opacity: 0.65; margin: 0;">
  OpenStack 기반 Private Cloud 플랫폼을 개발·운영하며,<br>
  GPU 인스턴스 관리, 네트워크, 로드밸런서 등 인프라 제어 기능을 직접 설계하고 구현합니다.<br>
  반복되는 운영 문제를 자동화로 해결하는 것에 관심이 많습니다.
</p>

---

## <i class="fas fa-thumbtack"></i> Pinned

<style>
  .ph-card {
    display: block; border: 1px solid var(--main-border-color, #e0e0e0);
    border-radius: 10px; padding: 1rem 1.15rem; text-decoration: none !important;
    color: inherit; transition: border-color 0.15s, transform 0.15s;
  }
  .ph-card:hover { border-color: var(--link-color); transform: translateY(-1px); }
  .ph-card .ph-title { font-size: 0.98rem; font-weight: 700; margin-bottom: 0.35rem; display: flex; align-items: center; gap: 0.45rem; }
  .ph-card .ph-title i { color: var(--link-color); opacity: 0.85; }
  .ph-card .ph-desc { font-size: 0.85rem; line-height: 1.65; opacity: 0.65; }
  @media (max-width: 480px) {
    .ph-grid { grid-template-columns: 1fr !important; }
  }
</style>


<div class="ph-grid" style="display: grid; grid-template-columns: repeat(2, 1fr); gap: 0.9rem; margin-bottom: 2rem;">
  <a class="ph-card" href="/octavia-manager/">
    <div class="ph-title"><i class="fas fa-random"></i> Octavia Manager</div>
    <div class="ph-desc">안 지워지는 로드밸런서 삭제부터 CLI 운영, 트러블슈팅까지 정리한 인터랙티브 운영 도구</div>
  </a>
  <a class="ph-card" href="/gpu-manager/">
    <div class="ph-title"><i class="fas fa-toolbox"></i> GPU Manager</div>
    <div class="ph-desc">GPU Passthrough·vGPU·MIG 개념 정리와, 직접 만든 gpu-manager CLI 소개 (GitHub 링크 포함)</div>
  </a>
  <a class="ph-card" href="{% post_url 2026-06-30-mdev-orphan-cause %}">
    <div class="ph-title"><i class="fas fa-ghost"></i> mdev orphan 자동화</div>
    <div class="ph-desc">수동 10분 걸리던 GPU mdev 판단 작업을, 도구화해서 비전문가도 바로 처리할 수 있게 만든 과정</div>
  </a>
  <a class="ph-card" href="{% post_url 2026-06-30-rack-topology %}">
    <div class="ph-title"><i class="fas fa-server"></i> Rack Topology 시각화</div>
    <div class="ph-desc">DB 설계부터 API까지 직접 구현한 데이터센터 랙/자원 시각화 프로젝트 — OpenStack 연동 가상 네트워크 토폴로지, 물리 네트워크 토폴로지 기능 포함</div>
  </a>
</div>

---

## 경력

<div style="margin: 1.5rem 0 2rem;">

  <!-- 회사 헤더 -->
  <div style="display: flex; align-items: baseline; justify-content: space-between; flex-wrap: wrap; gap: 0.15rem; margin-bottom: 0.2rem;">
    <div>
      <strong style="font-size: 1.05rem; letter-spacing: -0.01em;">오케스트로</strong>
      <a href="https://www.okestro.com/" target="_blank" style="font-size: 0.8rem; margin-left: 6px; opacity: 0.5;">↗</a>
    </div>
    <span style="font-size: 0.88rem; opacity: 0.45;">2022.08 ~ 재직중 &nbsp;<span id="duration"></span></span>
  </div>
  <p style="font-size: 0.9rem; line-height: 1.65; opacity: 0.5; margin: 0 0 1.25rem;">클라우드 인프라, 플랫폼, 운영 자동화, AI 기반 클라우드 관리 기술 개발 기업</p>

  <!-- 타임라인 -->
  <div style="border-left: 2px solid var(--main-border-color, #e0e0e0); padding-left: 1.25rem;">

    <!-- 콘트라베이스 최적화팀 -->
    <div style="position: relative; margin-bottom: 1.6rem;">
      <div style="position: absolute; left: -1.42rem; top: 0.4rem; width: 7px; height: 7px; border-radius: 50%; background: currentColor; opacity: 0.5;"></div>
      <div style="display: flex; align-items: baseline; flex-wrap: wrap; gap: 0.3rem; margin-bottom: 0.4rem;">
        <span style="font-size: 1rem; font-weight: 600; letter-spacing: -0.01em;">콘트라베이스 최적화팀</span>
        <span style="font-size: 0.82rem; opacity: 0.45;">2023.04 ~ 현재 &nbsp;<span id="dur1"></span></span>
      </div>
      <ul style="font-size: 0.95rem; line-height: 1.85; padding-left: 1rem; margin: 0; opacity: 0.8;">
        <li>OpenStack 기반 Private Cloud 관리 포탈 "Contrabass" 백엔드 개발 및 운영</li>
        <li>GPU Passthrough / MIG 인스턴스 생성 및 관리 기능 개발</li>
        <li>GPU Host, GPU Instance 모니터링 기능 개발</li>
        <li>GPU 운영 자동화 TUI 개발 (mdev orphan 탐지, MIG 프로파일 관리)</li>
        <li>Octavia 로드밸런서 장애 처리 자동화 도구 개발</li>
        <li>데이터센터 Rack Topology 시각화 기능 개발 (IPMI/SNMP)</li>
        <li>국방부 국방지능형플랫폼 GPU 모니터링 기능 개발 및 현장 기술 지원</li>
      </ul>
    </div>

    <!-- 플랫폼 공통 개발팀 -->
    <div style="position: relative; margin-bottom: 1.6rem;">
      <div style="position: absolute; left: -1.42rem; top: 0.4rem; width: 7px; height: 7px; border-radius: 50%; background: currentColor; opacity: 0.3;"></div>
      <div style="display: flex; align-items: baseline; flex-wrap: wrap; gap: 0.3rem; margin-bottom: 0.4rem;">
        <span style="font-size: 1rem; font-weight: 600; letter-spacing: -0.01em;">플랫폼 공통 개발팀</span>
        <span style="font-size: 0.82rem; opacity: 0.45;">2022.12 ~ 2023.03 (4개월)</span>
      </div>
      <ul style="font-size: 0.95rem; line-height: 1.85; padding-left: 1rem; margin: 0; opacity: 0.8;">
        <li>Samsung Cloud Platform PaaS(Kubernetes) 관리 포탈 Frontend 개발</li>
      </ul>
    </div>

    <!-- MLOps팀 -->
    <div style="position: relative;">
      <div style="position: absolute; left: -1.42rem; top: 0.4rem; width: 7px; height: 7px; border-radius: 50%; background: currentColor; opacity: 0.3;"></div>
      <div style="display: flex; align-items: baseline; flex-wrap: wrap; gap: 0.3rem; margin-bottom: 0.4rem;">
        <span style="font-size: 1rem; font-weight: 600; letter-spacing: -0.01em;">MLOps팀</span>
        <span style="font-size: 0.82rem; opacity: 0.45;">2022.08 ~ 2022.11 (3개월)</span>
      </div>
      <ul style="font-size: 0.95rem; line-height: 1.85; padding-left: 1rem; margin: 0; opacity: 0.8;">
        <li>전환형 인턴</li>
        <li>MLOps CI/CD 솔루션 "Trumpet.ai" 백엔드 및 프론트엔드 개발</li>
      </ul>
    </div>

  </div>
</div>

---

## Contact

<div style="margin: 1.25rem 0;">
  <div style="margin-bottom: 0.75rem;">
    <div style="font-size: 0.75rem; font-weight: 700; opacity: 0.35; text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 0.15rem;">GitHub</div>
    <a href="https://github.com/bbdawn" style="font-size: 0.95rem;">github.com/bbdawn</a>
  </div>
  <div>
    <div style="font-size: 0.75rem; font-weight: 700; opacity: 0.35; text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 0.15rem;">Email</div>
    <span style="font-size: 0.95rem;">leejoohyun_my@naver.com</span>
  </div>
</div>

<script>
  function calcDuration() {
    const now = new Date();
    function diff(startDate) {
      const start = new Date(startDate);
      let years = now.getFullYear() - start.getFullYear();
      let months = now.getMonth() - start.getMonth();
      let days = now.getDate() - start.getDate();
      if (days < 0) { months--; days += new Date(now.getFullYear(), now.getMonth(), 0).getDate(); }
      if (months < 0) { years--; months += 12; }
      return { years, months };
    }
    const d0 = diff('2022-08-22');
    document.getElementById('duration').textContent = `${d0.years}년 ${d0.months}개월 재직중`;
    const d1 = diff('2023-04-03');
    document.getElementById('dur1').textContent = `(${d1.years}년 ${d1.months}개월)`;
  }
  calcDuration();
</script>
