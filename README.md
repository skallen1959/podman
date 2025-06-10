# My Podman Adventures 🐧

Welcome to my collection of Podman experiments and real-world deployments! As a system administrator since 1991 and Linux enthusiast since 1993, I've been exploring the world of rootless containers with Podman from both a curiosity and security perspective.

## Why Podman?

After years of working with various container technologies, I've become increasingly interested in Podman for several compelling reasons:

- **Rootless by design** - Enhanced security through privilege separation
- **Docker-compatible** - Easy migration from existing Docker workflows  
- **Systemd integration** - Native support for running containers as system services
- **No daemon required** - Eliminates single points of failure
- **Pod support** - Kubernetes-style pod management without the complexity

## Current Projects

### 🦊 [GitLab CE on Debian 12](debian/gitlab/README.md)
A complete production-ready GitLab installation using rootless Podman with:
- **Two deployment options**: Corporate (with LDAP/AD integration) and Home Lab (simple setup)
- **Security hardening**: nftables firewall, SSH hardening, automatic updates
- **SSL/TLS**: Let's Encrypt integration with automatic renewal
- **High availability storage**: Originally designed for CephFS, adapted for standard filesystems
- **Backup strategy**: Automated backup solutions
- **Reverse proxy**: Nginx frontend for standard port access

Perfect for organizations wanting to self-host GitLab without the overhead of Docker or Kubernetes.

## Upcoming Adventures

I'm planning to expand this collection with several practical services, each implemented across different Linux distributions:

### 📧 **SMTP Forwarding to Google Workspace** 
Secure email relay setup using Podman for hybrid email environments

### 🔔 **ntfy.sh with LDAP Integration**
Self-hosted notification service with Active Directory authentication

### 🖥️ **RustDesk Server** 
Open-source remote desktop solution for secure internal access

### 🐧 **Multi-Distribution Support**
Each service will be documented for:
- 🏔️ **Rocky Linux / AlmaLinux** - Enterprise Linux with SELinux considerations  
- 🏹 **Arch Linux** - Bleeding edge implementations
- 📦 **Debian** - Stable, production-ready deployments

### 🔧 **Advanced Topics**
- Pod orchestration and inter-container communication
- Migration strategies from Docker Compose
- Backup and disaster recovery patterns

## Background

Having worked with everything from early Linux distributions to modern container orchestration, I believe in sharing practical, battle-tested solutions. Too many experienced system administrators keep their knowledge locked away - this repository is my contribution to changing that culture.

## Contributing

Found an issue? Have a suggestion? Want to share your own Podman adventure? 

- **Issues**: Report problems or request new guides
- **Pull Requests**: Improvements and corrections are always welcome
- **Discussions**: Share your own experiences and variations

## License

All guides and configurations are provided under the MIT License - use them freely in your own environments!

---

*"The best system administrators are those who document their solutions and share their knowledge."*
