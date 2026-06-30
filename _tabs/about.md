---
icon: fas fa-info-circle
order: 6
---

# 이주현

개발도 하고 인프라도 아는 클라우드 백엔드 개발자입니다.

OpenStack 기반 Private Cloud 플랫폼을 개발·운영하며, GPU 인스턴스 관리, 네트워크, 로드밸런서 등 인프라 제어 기능을 직접 설계하고 구현합니다. 반복되는 운영 문제를 자동화로 해결하는 것에 관심이 많습니다.

---

## 경력

### <a href="https://www.okestro.com/" target="_blank">오케스트로 ↗</a>

`2022.08.22 ~ 재직중` &nbsp; <span id="duration" class="text-muted small"></span>

**콘트라베이스 최적화팀** `2023.04 ~ 현재`

- OpenStack 기반 Private Cloud 관리 포탈 "Contrabass" 백엔드 개발 및 운영
- GPU Passthrough / MIG 인스턴스 관리 및 모니터링 기능 개발
- GPU 운영 자동화 TUI 개발 (mdev orphan 탐지, MIG 프로파일 관리)
- Octavia 로드밸런서 장애 처리 자동화 TUI 개발
- 데이터센터 Rack Topology 시각화 기능 개발 (IPMI/SNMP)
- 국방부 국방지능형플랫폼 GPU 모니터링 기능 개발 및 현장 기술 지원

**플랫폼 공통 개발팀** `2022.12 ~ 2023.03`

- Samsung Cloud Platform PaaS(Kubernetes) 관리 포탈 화면 개발

**MLOps팀** `2022.08 ~ 2022.11`

- MLOps CI/CD 솔루션 "Trumpet.ai" 백엔드 및 프론트엔드 개발

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
    const start = new Date('2022-08-22');
    const now = new Date();

    let years = now.getFullYear() - start.getFullYear();
    let months = now.getMonth() - start.getMonth();
    let days = now.getDate() - start.getDate();

    if (days < 0) {
      months--;
      days += new Date(now.getFullYear(), now.getMonth(), 0).getDate();
    }
    if (months < 0) {
      years--;
      months += 12;
    }

    document.getElementById('duration').textContent = `(${years}년 ${months}개월 ${days}일 재직중)`;
  }
  calcDuration();
</script>
