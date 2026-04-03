---
title: Projects
date: 2026-04-01
menu:
    main:
        weight: -70
        params:
            icon: infinity
---

<div class="project-list">

<a href="/page/project/ecr-replication/" class="project-card">
  <div class="project-card-inner">
    <div class="project-icon">
      <img src="/icons/aws/ecr.svg" alt="Amazon ECR icon" width="40" height="40">
    </div>
    <div class="project-info">
      <h3>Cross-Account ECR Replication</h3>
      <p>EventBridge + Lambda를 활용한 Cross-Account ECR 태그 및 Lifecycle 정책 자동 복제 아키텍처</p>
      <div class="project-tags">
        <span>AWS</span><span>Lambda</span><span>EventBridge</span><span>ECR</span>
      </div>
    </div>
    <div class="project-arrow">&rarr;</div>
  </div>
</a>

</div>

<style>
.project-list { display: flex; flex-direction: column; gap: 16px; }
.project-card {
  display: block;
  text-decoration: none !important;
  color: inherit !important;
  background: var(--card-background, #fff);
  border: 1px solid var(--card-separator-color, #e0e0e0);
  border-radius: 12px;
  padding: 20px;
  transition: transform 0.2s, box-shadow 0.2s;
}
.project-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.12);
}
.project-card-inner {
  display: flex;
  align-items: center;
  gap: 16px;
}
.project-icon { flex-shrink: 0; }
.project-icon img { display: block; border-radius: 8px; }
.project-info { flex: 1; }
.project-info h3 { margin: 0 0 6px; font-size: 1.1rem; }
.project-info p { margin: 0 0 8px; font-size: 0.85rem; opacity: 0.7; }
.project-tags { display: flex; gap: 6px; flex-wrap: wrap; }
.project-tags span {
  background: rgba(255,153,0,0.15);
  color: #FF9900;
  font-size: 0.72rem;
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: 600;
}
.project-arrow { font-size: 1.5rem; opacity: 0.3; flex-shrink: 0; }
.project-card:hover .project-arrow { opacity: 1; color: #FF9900; }

@media (max-width: 600px) {
  .project-card-inner { flex-direction: column; align-items: flex-start; }
  .project-arrow { display: none; }
}
</style>
