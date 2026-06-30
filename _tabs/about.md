---
icon: fas fa-info-circle
order: 6
---

# 이주현

개발도 하고 인프라도 아는 클라우드 백엔드 개발자입니다.

OpenStack 기반 Private Cloud 플랫폼을 개발·운영하며, GPU 인스턴스 관리, 네트워크, 로드밸런서 등 인프라 제어 기능을 직접 설계하고 구현합니다. 반복되는 운영 문제를 자동화로 해결하는 것에 관심이 많습니다.

---

## 경력

<div style="margin: 1.5rem 0 2rem;">

  <!-- 회사 헤더 -->
  <div style="display: flex; align-items: baseline; justify-content: space-between; margin-bottom: 0.25rem;">
    <div>
      <strong style="font-size: 1.05rem;">오케스트로</strong>
      <a href="https://www.okestro.com/" target="_blank" style="font-size: 0.75rem; margin-left: 6px; opacity: 0.6;">↗</a>
    </div>
    <span style="font-size: 0.8rem; opacity: 0.55;">2022.08 ~ 재직중 &nbsp;<span id="duration"></span></span>
  </div>
  <p style="font-size: 0.82rem; opacity: 0.55; margin: 0 0 1.25rem;">AI · 클라우드 솔루션 전문 기업 — 클라우드 인프라, 플랫폼, 운영 자동화, AI 기반 클라우드 관리 기술 개발</p>

  <!-- 타임라인 -->
  <div style="border-left: 2px solid #ddd; padding-left: 1.25rem;">

    <!-- 콘트라베이스 최적화팀 -->
    <div style="position: relative; margin-bottom: 1.5rem;">
      <div style="position: absolute; left: -1.4rem; top: 0.35rem; width: 8px; height: 8px; border-radius: 50%; background: #888;"></div>
      <div style="display: flex; align-items: baseline; gap: 0.5rem; margin-bottom: 0.4rem;">
        <span style="font-weight: 600; font-size: 0.9rem;">콘트라베이스 최적화팀</span>
        <span style="font-size: 0.78rem; opacity: 0.55;">2023.04 ~ 현재 &nbsp;<span id="dur1"></span></span>
      </div>
      <ul style="font-size: 0.85rem; padding-left: 1rem; margin: 0; line-height: 1.8;">
        <li>OpenStack 기반 Private Cloud 관리 포탈 "Contrabass" 백엔드 개발 및 운영</li>
        <li>GPU Passthrough / MIG 인스턴스 관리 및 모니터링 기능 개발</li>
        <li>GPU 운영 자동화 TUI 개발 (mdev orphan 탐지, MIG 프로파일 관리)</li>
        <li>Octavia 로드밸런서 장애 처리 자동화 TUI 개발</li>
        <li>데이터센터 Rack Topology 시각화 기능 개발 (IPMI/SNMP)</li>
        <li>국방부 국방지능형플랫폼 GPU 모니터링 기능 개발 및 현장 기술 지원</li>
      </ul>
    </div>

    <!-- 플랫폼 공통 개발팀 -->
    <div style="position: relative; margin-bottom: 1.5rem;">
      <div style="position: absolute; left: -1.4rem; top: 0.35rem; width: 8px; height: 8px; border-radius: 50%; background: #bbb;"></div>
      <div style="display: flex; align-items: baseline; gap: 0.5rem; margin-bottom: 0.4rem;">
        <span style="font-weight: 600; font-size: 0.9rem;">플랫폼 공통 개발팀</span>
        <span style="font-size: 0.78rem; opacity: 0.55;">2022.12 ~ 2023.03 (4개월)</span>
      </div>
      <ul style="font-size: 0.85rem; padding-left: 1rem; margin: 0; line-height: 1.8;">
        <li>Samsung Cloud Platform PaaS(Kubernetes) 관리 포탈 화면 개발</li>
      </ul>
    </div>

    <!-- MLOps팀 -->
    <div style="position: relative;">
      <div style="position: absolute; left: -1.4rem; top: 0.35rem; width: 8px; height: 8px; border-radius: 50%; background: #bbb;"></div>
      <div style="display: flex; align-items: baseline; gap: 0.5rem; margin-bottom: 0.4rem;">
        <span style="font-weight: 600; font-size: 0.9rem;">MLOps팀</span>
        <span style="font-size: 0.78rem; opacity: 0.55;">2022.08 ~ 2022.11 (3개월)</span>
      </div>
      <ul style="font-size: 0.85rem; padding-left: 1rem; margin: 0; line-height: 1.8;">
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

## Contact

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
