# Replicated Distributed Apps

Claude Code skills for building and distributing apps with the Replicated platform — distilled from real-world experience.

## Skills

| Skill | Description |
|-------|------------|
| [replicated-ci-setup](skills/replicated-ci-setup/SKILL.md) | CI/CD with Replicated CLI — workflows, RBAC, CMX testing |
| [replicated-image-proxy](skills/replicated-image-proxy/SKILL.md) | Custom domain image proxying for app and subchart images |
| [replicated-sdk-integration](skills/replicated-sdk-integration/SKILL.md) | SDK subchart — license gating, metrics, updates, validity |
| [replicated-release-management](skills/replicated-release-management/SKILL.md) | release-please + Replicated release flow, semver, promotion |
| [replicated-helm-patterns](skills/replicated-helm-patterns/SKILL.md) | CRD ordering, webhook timing, KOTS manifests, BYO pattern |

## Installation

```bash
# Clone the repo
git clone https://github.com/jmboby/replicated-admin-skills.git

# Install as a Claude Code plugin from local directory
claude plugin add /path/to/replicated-admin-skills
```

## Contributing

These skills are based on lessons learned from building a Replicated-distributed app. If you've discovered patterns or gotchas not covered here, PRs are welcome.
