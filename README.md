<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://cdn.simpleicons.org/python/white" />
    <img alt="Python Logo" src="https://cdn.simpleicons.org/python/black" width="60" />
  </picture>
</p>

<h1 align="center">Awesome Python</h1>

<p align="center">
  <a href="https://awesome.re"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=awesomelists" /><img alt="awesome" src="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=awesomelists" /></picture></a>
  <a href="https://www.python.org/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/Python-3.10+.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=python" /><img alt="Python" src="https://shieldcn.dev/badge/Python-3.10+.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=python" /></picture></a>
  <a href="LICENSE"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="license" src="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
  <a href="https://github.com/bhagirath00/awesome-python"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/views/repo/bhagirath00/awesome-python.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="views" src="https://shieldcn.dev/views/repo/bhagirath00/awesome-python.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
</p>

<p align="center">
  <em>A curated, professional-grade guide to mastering Python for DevOps, cloud automation, systems engineering, and infrastructure orchestration.</em>
</p>

---

## Table of Contents

- [Introduction](#introduction)
- [Why Python for DevOps?](#why-python-for-devops)
- [Learning Modules](#learning-modules)
  - [Phase 1: Foundations & Data Serialization](#phase-1-foundations--data-serialization)
  - [Phase 2: Systems, Networking & Cloud Automation](#phase-2-systems-networking--cloud-automation)
  - [Phase 3: Containers, Kubernetes & CI/CD Pipelines](#phase-3-containers-kubernetes--cicd-pipelines)
  - [Phase 4: Observability, IaC, Security & Best Practices](#phase-4-observability-iac-security--best-practices)
- [Resources](#resources)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

---

## Introduction

Welcome to **Awesome Python**! This repository serves as an exhaustive, hands-on architectural roadmap and reference manual designed for site reliability engineers (SREs), DevOps practitioners, and platform engineers.

> **"Simple is better than complex. Readability counts."** — The Zen of Python (Tim Peters)

---

## Why Python for DevOps?

Python is the multi-purpose glue of modern infrastructure engineering and cloud automation:

| Capability | Importance in DevOps |
| :--- | :--- |
| **Readable & Maintainable** | Expressive syntax lowers maintenance friction for shared operational tooling and CLI scripts |
| **Ubiquitous Cloud SDKs** | First-class official SDKs for AWS (Boto3), Google Cloud, Azure, Kubernetes, and Docker |
| **Rich Ecosystem** | Massive library support for REST APIs, SSH, YAML/JSON parsing, cryptography, and testing |
| **Glue Between Systems** | Integrates CI/CD runners, IaC state parsers, chatops bots, and monitoring pipelines effortlessly |

---

## Learning Modules

Organized into 14 comprehensive modules covering 54 chapters and 303 production-grade guides.

### Phase 1: Foundations & Data Serialization

| Module | Description |
| :--- | :--- |
| [**Module 01 - Introduction to Python for DevOps**](./Module%2001%20-%20Introduction%20to%20Python%20for%20DevOps/) | Why Python for DevOps, Python vs Bash vs Go, virtual environments (`venv`), and interpreter mechanics |
| [**Module 02 - Python Fundamentals**](./Module%2002%20-%20Python%20Fundamentals/) | Data types, lists, dictionaries, sets, control flow, functions, modules, error handling, and file I/O |
| [**Module 03 - Working with Data Formats**](./Module%2003%20-%20Working%20with%20Data%20Formats/) | Parsing and manipulating structured configuration data: JSON, YAML (PyYAML), XML, CSV, TOML, and INI |

### Phase 2: Systems, Networking & Cloud Automation

| Module | Description |
| :--- | :--- |
| [**Module 04 - OS and System Automation**](./Module%2004%20-%20OS%20and%20System%20Automation/) | Operating system interactions via `os`, `sys`, process management with `subprocess`, and `shutil` |
| [**Module 05 - Networking with Python**](./Module%2005%20-%20Networking%20with%20Python/) | Low-level sockets, HTTP clients (`requests`), REST APIs, and automated remote SSH with Paramiko |
| [**Module 06 - AWS Automation with Python**](./Module%2006%20-%20AWS%20Automation%20with%20Python/) | Cloud engineering with Boto3: EC2 lifecycle management, S3 operations, IAM policy audits, and CloudWatch |

### Phase 3: Containers, Kubernetes & CI/CD Pipelines

| Module | Description |
| :--- | :--- |
| [**Module 07 - Docker Automation with Python**](./Module%2007%20-%20Docker%20Automation%20with%20Python/) | Docker SDK for Python: container lifecycles, dynamic image builds, pruning, and Compose automation |
| [**Module 08 - Kubernetes Automation with Python**](./Module%2008%20-%20Kubernetes%20Automation%20with%20Python/) | Official Kubernetes client: managing Pods, Deployments, Services, CRDs, watch APIs, and operator patterns |
| [**Module 09 - CI-CD Automation with Python**](./Module%2009%20-%20CI-CD%20Automation%20with%20Python/) | Git automation with GitPython, GitHub REST API, Jenkins REST API automation, and pipeline scripting |

### Phase 4: Observability, IaC, Security & Best Practices

| Module | Description |
| :--- | :--- |
| [**Module 10 - Monitoring and Logging with Python**](./Module%2010%20-%20Monitoring%20and%20Logging%20with%20Python/) | Standard `logging` module, Prometheus client metrics, Slack/PagerDuty alerts, and log parsing |
| [**Module 11 - Infrastructure as Code with Python**](./Module%2011%20-%20Infrastructure%20as%20Code%20with%20Python/) | Terraform automation (`python-terraform`), Ansible runner automation, and Pulumi Python SDK |
| [**Module 12 - Security Automation with Python**](./Module%2012%20-%20Security%20Automation%20with%20Python/) | HashiCorp Vault secrets integration (Hvac), SAST security scanning (Bandit), and cryptography |
| [**Module 13 - Testing for DevOps with Python**](./Module%2013%20-%20Testing%20for%20DevOps%20with%20Python/) | Unit testing with `pytest`, mocking external APIs with `unittest.mock`, and infrastructure test suites |
| [**Module 14 - Python Scripting Best Practices**](./Module%2014%20-%20Python%20Scripting%20Best%20Practices/) | PEP 8 style, CLI design (`argparse`, `click`, `rich`), profiling, packaging, and production reliability |

---

## Resources

### Official Documentation

| Resource | Link |
| :--- | :--- |
| Python Documentation | [docs.python.org/3](https://docs.python.org/3/) |
| Python Package Index (PyPI) | [pypi.org](https://pypi.org/) |
| PEP 8 Style Guide | [peps.python.org/pep-0008](https://peps.python.org/pep-0008/) |
| Real Python | [realpython.com](https://realpython.com/) |

### Essential DevOps Python Libraries

| Library | Purpose | Link |
| :--- | :--- | :--- |
| `requests` | HTTP Client for REST APIs | [requests.readthedocs.io](https://requests.readthedocs.io/) |
| `boto3` | AWS SDK for Python | [boto3.amazonaws.com](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) |
| `kubernetes` | Official Kubernetes Python Client | [github.com/kubernetes-client/python](https://github.com/kubernetes-client/python) |
| `docker` | Docker Engine API SDK | [docker-py.readthedocs.io](https://docker-py.readthedocs.io/) |
| `paramiko` | SSH2 protocol library | [paramiko.org](https://www.paramiko.org/) |
| `click` / `rich` | CLI creation & terminal formatting | [click.palletsprojects.com](https://click.palletsprojects.com/) |
| `pytest` | Testing framework | [docs.pytest.org](https://docs.pytest.org/) |

---

## Contributing

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting a Pull Request.

---

## Security

Please review our [Security Policy](SECURITY.md) for script hardening guidelines, secret handling, and vulnerability reporting.

---

## License

This educational content is open source and available under the **MIT License**. See [LICENSE](LICENSE) for details.

<p align="center">
  <strong>Happy Pythoning!</strong>
</p>
