# Raspberry Pi Cluster Home Lab

A comprehensive resource repository documenting the journey of building, configuring, and managing a Raspberry Pi cluster for learning, experimentation, and self-hosting.

## Overview

This repository contains configuration files, scripts, documentation, and resources related to building a home lab Raspberry Pi cluster. It serves as both a reference for my blog series and a practical resource for others interested in similar projects.

## What's Inside

### Documentation ([docs/](docs/))

Detailed guides and tutorials covering various aspects of the cluster build and related infrastructure:

- [How to Set Up a Hugo Blog on Cloudflare Pages](docs/How%20to%20Set%20Up%20a%20Hugo%20Blog%20on%20Cloudflare%20Pages.md) - Complete guide for setting up a technical blog to document your homelab journey

### Configuration Files ([configs/](configs/))

Ready-to-use configuration files for various services and components:

- Kubernetes manifests
- Docker Compose files
- Network configurations
- Service configurations
- *Coming soon as the cluster build progresses*

### Scripts ([scripts/](scripts/))

Automation scripts and utilities for cluster management:

- Installation and setup scripts
- Maintenance and monitoring tools
- Backup and recovery utilities
- *Coming soon as the cluster build progresses*

## Project Goals

This Raspberry Pi cluster project aims to:

- **Learn by Doing**: Hands-on experience with distributed systems, container orchestration, and infrastructure management
- **Self-Hosting**: Run personal services and applications on self-managed infrastructure
- **Experimentation**: Safe environment to test new technologies and configurations
- **Documentation**: Share knowledge and help others on similar journeys
- **Cost-Effective**: Build powerful infrastructure using affordable single-board computers

## Planned Architecture

The cluster will include:

- **Hardware**: Multiple Raspberry Pi nodes for compute and storage
- **Orchestration**: Kubernetes (likely K3s for lightweight deployment)
- **Networking**: Custom network configuration with VLANs and proper segmentation
- **Storage**: Distributed storage solution for persistent data
- **Monitoring**: Prometheus, Grafana, and other observability tools
- **Services**: Self-hosted applications and services

## Getting Started

This repository is designed to be both a learning resource and a practical reference. You can:

1. **Follow Along**: Read my blog at [jondepalma.com](https://jondepalma.com), and this repo as I build and document the cluster
2. **Use Configurations**: Adapt configuration files for your own projects
3. **Run Scripts**: Utilize automation scripts for common tasks
4. **Learn**: Understand concepts and implementation details through detailed guides

## Repository Structure

```
picluster/
├── docs/               # Documentation and guides
├── configs/            # Configuration files for services
├── scripts/            # Automation and utility scripts
|── LICENSE             # MIT LICENSEs
└── README.md           # This file
```

## Blog Series

This repository supports an ongoing blog series documenting the entire build process, from hardware selection to running production services. Follow along at [jondepalma.com](https://jondepalma.com).

### Planned Topics

- **Phase 1:** Foundation - Hardware, K3s, and core networking
- **Phase 2:** Networking & Security - VPNs, tunnels, and SSL
- **Phase 3:** Ghost Blog - Migrate and self-hosting this very blog!
- **Phase 4:** Development Environment - Databases and tools
- **Phase 5:** Production Environment - Real applications
- **Phase 6:** Monitoring & CI/CD - Keeping it all running

## Contributing

While this is primarily a personal learning project, suggestions and improvements are welcome! Feel free to:

- Open issues for questions or suggestions
- Submit pull requests for corrections or improvements
- Share your own experiences and learnings

## Resources

### Hardware

- 3x Raspberry Pi 5 units with 256GB SD
    - 1 controller, 2 worker nodes
- 1x Raspberry Pi M.2 HAT with 1TB nVME storage for controller
- 8-port Gigabit Switch (Netgear GS308E)
- Power supply and management

### Software Stack

- **OS**: Raspberry Pi OS / Ubuntu Server
- **Container Runtime**: Docker / GHCR
- **Orchestration**: K3s / Kubernetes
- **Blogging**: Hugo (to be replaced by Ghost)
- **Hosting/Ingress**: Cloudflare, Tailscale, Traefik, MetalLB
- **Monitoring**: Beszel
- **Storage**: Node storage and NAS

### Useful Links

- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [K3s Documentation](https://docs.k3s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Documentation](https://docs.docker.com/)

## Current Status

**Project Status**: Initial Setup Phase

- [x] Repository created
- [x] Documentation structure established
- [x] Hardware procurement
- [ ] Foundation - Hardware, K3s, and core networking
- [ ] Networking & Security - VPNs, tunnels, and SSL
- [ ] Ghost Blog - Migrate and self-hosting this very blog!
- [ ] Development Environment - Databases and tools
- [ ] Production Environment - Real applications
- [ ] Monitoring & CI/CD - Keeping it all running

*This is a living document that will be updated as the project progresses.*

## License

This project is open source and available under the [MIT License](LICENSE).

## Contact

Questions, suggestions, or want to share your own cluster build? Reach out via [jondepalma.com](https://jondepalma.com)

- **Blog**: [jondepalma.com](https://jondepalma.com)
- **GitHub Issues**: [Issues page](../../issues)
- **X**: [\@jondepalma](https://x.com/jondepalma)

---

**Note**: This is an active learning project. Configurations and documentation will evolve as I learn and improve the setup. All content is provided as-is for educational purposes.

*Last updated: November 2025*
