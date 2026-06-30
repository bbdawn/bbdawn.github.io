---
icon: fas fa-info-circle
order: 6
---

# 이주현

개발도 하고 인프라도 아는 클라우드 백엔드 개발자입니다.

OpenStack 기반 Private Cloud 플랫폼을 개발·운영하며, GPU 인스턴스 관리, 네트워크, 로드밸런서 등 인프라 제어 기능을 직접 설계하고 구현합니다. 반복되는 운영 문제를 자동화로 해결하는 것에 관심이 많습니다.

---

## 경력

<div class="card mb-4" style="border-radius: 12px; border: 1px solid var(--card-border-color);">
  <div class="card-body p-4">

    <!-- 회사 헤더 -->
    <div class="d-flex align-items-start justify-content-between mb-2">
      <div>
        <h4 class="mb-1" style="font-weight: 700;">
          오케스트로
          <a href="https://www.okestro.com/" target="_blank" style="font-size: 0.75rem; font-weight: 400; margin-left: 8px; vertical-align: middle;">↗ 바로가기</a>
        </h4>
        <p class="text-muted mb-2" style="font-size: 0.875rem;">AI · 클라우드 솔루션 전문 기업으로 클라우드 인프라, 플랫폼, 운영 자동화, 인공지능 기반 클라우드 관리 기술 등을 개발하고 제공하는 회사입니다.</p>
        <span class="badge rounded-pill" style="background-color: var(--link-color); font-size: 0.75rem; font-weight: 500;">2022.08.22 ~ 재직중 &nbsp;<span id="duration"></span></span>
      </div>
    </div>

    <hr style="margin: 1rem 0;">

    <!-- 콘트라베이스 최적화팀 -->
    <div class="mb-4">
      <div class="d-flex align-items-center gap-2 mb-2">
        <span style="font-weight: 600; font-size: 0.95rem;">콘트라베이스 최적화팀</span>
        <span class="text-muted" style="font-size: 0.8rem;">2023.04 ~ 현재</span>
        <span class="text-muted" style="font-size: 0.8rem;" id="dur1"></span>
      </div>
      <ul class="mb-0" style="font-size: 0.875rem; padding-left: 1.2rem;">
        <li>OpenStack 기반 Private Cloud 관리 포탈 "Contrabass" 백엔드 개발 및 운영</li>
        <li>GPU Passthrough / MIG 인스턴스 관리 및 모니터링 기능 개발</li>
        <li>GPU 운영 자동화 TUI 개발 (mdev orphan 탐지, MIG 프로파일 관리)</li>
        <li>Octavia 로드밸런서 장애 처리 자동화 TUI 개발</li>
        <li>데이터센터 Rack Topology 시각화 기능 개발 (IPMI/SNMP)</li>
        <li>국방부 국방지능형플랫폼 GPU 모니터링 기능 개발 및 현장 기술 지원</li>
      </ul>
    </div>

    <!-- 플랫폼 공통 개발팀 -->
    <div class="mb-4">
      <div class="d-flex align-items-center gap-2 mb-2">
        <span style="font-weight: 600; font-size: 0.95rem;">플랫폼 공통 개발팀</span>
        <span class="text-muted" style="font-size: 0.8rem;">2022.12 ~ 2023.03</span>
        <span class="text-muted" style="font-size: 0.8rem;">(4개월)</span>
      </div>
      <ul class="mb-0" style="font-size: 0.875rem; padding-left: 1.2rem;">
        <li>Samsung Cloud Platform PaaS(Kubernetes) 관리 포탈 화면 개발</li>
      </ul>
    </div>

    <!-- MLOps팀 -->
    <div>
      <div class="d-flex align-items-center gap-2 mb-2">
        <span style="font-weight: 600; font-size: 0.95rem;">MLOps팀</span>
        <span class="text-muted" style="font-size: 0.8rem;">2022.08 ~ 2022.11</span>
        <span class="text-muted" style="font-size: 0.8rem;">(3개월)</span>
      </div>
      <ul class="mb-0" style="font-size: 0.875rem; padding-left: 1.2rem;">
        <li>MLOps CI/CD 솔루션 "Trumpet.ai" 백엔드 및 프론트엔드 개발</li>
      </ul>
    </div>

  </div>
</div>

---

## 기술 스택

**Backend**
`Java` `Spring Boot` `Go`

**Infra**
`OpenStack` `Terraform` `Ansible` `Linux`

**Monitoring**
`Prometheus` `DCGM Exporter` `Grafana`

**GPU**
`NVIDIA A100 / H100 / B300` `GPU Passthrough` `MIG`

**협업**
`Git` `Jira` `Confluence`

---

## 연락처

- **GitHub**: [github.com/bbdawn](https://github.com/bbdawn)
- **Email**: leejoohyun_my@naver.com

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
