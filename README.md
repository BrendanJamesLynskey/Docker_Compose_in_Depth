# 🧩 Docker Compose in Depth

An interactive Reveal.js presentation on Docker Compose — services, networks, volumes, profiles, watch mode, multi-environment configs, scaling, and YAML anchors.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Docker_Compose_in_Depth/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | What is Docker Compose? | Declarative multi-container apps in YAML |
| 02 | Compose File Structure | Services, networks, and volumes top-level keys |
| 03 | Service Configuration Deep Dive | Container, networking, runtime options |
| 04 | Build vs Image | Pulling images or building from Dockerfile |
| 05 | Environment Variables & .env Files | Inline, env_file, shell, and priority order |
| 06 | depends_on & Healthchecks | Ordering startup with service_healthy conditions |
| 07 | Networking in Compose | Default bridge, custom networks, internal networks |
| 08 | Volume Management in Compose | Named volumes, bind mounts, tmpfs |
| 09 | Compose Profiles | Selectively start services by profile |
| 10 | Compose Watch (Hot Reload) | sync, rebuild, and sync+restart actions |
| 11 | Multi-Environment Configs | Base plus override files for dev/staging/prod |
| 12 | Scaling Services | CLI scaling and deploy.replicas for load balancing |
| 13 | Commands Cheat Sheet | up, down, logs, exec, build, watch |
| 14 | Production vs Development Configs | Build, volumes, ports, restart, resources |
| 15 | Docker Compose vs Kubernetes | Scope, scaling, learning curve, best fit |
| 16 | Compose Extensions & YAML Anchors | x- fields, anchors, aliases, merge keys |
| 17 | Summary & Further Reading | Key takeaways and official docs |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Docker Compose Documentation](https://docs.docker.com/compose/) · [Compose Specification](https://docs.docker.com/compose/compose-file/) · [Compose Profiles](https://docs.docker.com/compose/profiles/) · [Compose Watch](https://docs.docker.com/compose/file-watch/)

## License

Educational use. Code examples provided as-is.
