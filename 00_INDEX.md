# Formation CI/CD Avancée - GitLab CI + Nexus Repository Manager

> **Public cible** : Ingénieurs DevSecOps, DevOps, Backend Engineers ayant une expérience solide en développement et une première exposition aux pipelines CI/CD.
> **Prérequis** : Git maîtrisé, Docker & Kubernetes connus, Linux aisé, notions réseau de base.
> **Durée estimée** : 4–5 jours intensifs (env. 35h)

---

## 🗂️ Structure de la formation

| # | Module | Thèmes clés | Durée est. |
|---|--------|-------------|------------|
| [01](./01_GITLAB_ARCHITECTURE.md) | **Architecture GitLab & Runners** | GitLab topology, Runner types, executor strategies, registration, autoscaling | 4h |
| [02](./02_GITLAB_CI_FONDAMENTAUX.md) | **GitLab CI - Fondamentaux avancés** | `.gitlab-ci.yml` deep dive, stages, jobs, rules, needs, DAG, cache, artifacts | 5h |
| [03](./03_GITLAB_CI_AVANCE.md) | **GitLab CI - Patterns avancés** | Templates, includes, extends, matrices, DAST/SAST intégré, secrets management | 5h |
| [04](./04_NEXUS_ARCHITECTURE.md) | **Nexus Repository Manager - Architecture** | Types de repos, proxying, groupes, formats, Blob Store, HA | 3h |
| [05](./05_NEXUS_INTEGRATION.md) | **Nexus - Intégration CI/CD** | Maven, pip, npm, Docker, Helm dans les pipelines, cleanup policies | 4h |
| [06](./06_SECURITE_DEVSECOPS.md) | **Sécurité & DevSecOps** | SAST, DAST, SCA, container scanning, IaC scanning, secret detection, compliance | 5h |
| [07](./07_PIPELINE_PATTERNS.md) | **Pipeline Patterns & Architecture** | Multi-project, parent-child, environments, deployments, rollback, GitOps | 5h |
| [08](./08_OBSERVABILITE_PERFORMANCE.md) | **Observabilité & Performance** | Métriques pipelines, optimisation, monitoring GitLab, Nexus metrics, tracing | 3h |
| [09](./09_LABS_PRATIQUES.md) | **Labs Pratiques** | 8 labs progressifs, de la mise en place à un pipeline production-grade complet | 6h |

---

## 🎯 Objectifs pédagogiques

À l'issue de cette formation, le participant sera capable de :

- Concevoir et déployer une infrastructure GitLab CI hautement disponible avec autoscaling des runners
- Écrire des pipelines complexes tirant parti du DAG, des matrices et des templates réutilisables
- Administrer Nexus Repository Manager en production (multi-format, proxying, sécurité)
- Intégrer des scans de sécurité complets (SAST, DAST, SCA, Container Scanning) dans les pipelines
- Implémenter des stratégies de déploiement avancées (blue/green, canary, GitOps)
- Optimiser les performances et la fiabilité des pipelines CI/CD
- Diagnostiquer et résoudre les problèmes courants en production

---

## 🧱 Stack technique couverte

```
GitLab CE/EE 16.x+      Nexus Repository Manager 3.x OSS/Pro
GitLab Runners           Maven / npm / pip / Docker / Helm
Docker / Podman          Kubernetes (GKE, EKS, bare metal)
Terraform (IaC scan)     Trivy / Grype / Semgrep / OWASP ZAP
Vault (secrets)          Prometheus + Grafana (observabilité)
```

---

## 📐 Conventions utilisées dans ce cours

- `📋 Concept` - Notion théorique à retenir
- `⚙️ Config` - Extrait de configuration commenté
- `🔬 Lab` - Exercice pratique
- `⚠️ Piège` - Erreur fréquente à éviter
- `🔒 Sécurité` - Point de vigilance sécurité
- `🚀 Perf` - Optimisation performance
- `💡 Tip` - Bonne pratique ou astuce

---

## 🔗 Ressources complémentaires

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Nexus Repository OSS Docs](https://help.sonatype.com/en/nexus-repository-oss.html)
- [GitLab Security Scanning](https://docs.gitlab.com/ee/user/application_security/)
- [Nexus IQ / Lifecycle](https://help.sonatype.com/en/nexus-iq-server.html)
