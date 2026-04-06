# 🔐 AI Security Showcase

> Hands-on research into AI security vulnerabilities — systematically implementing proof-of-concept projects based on the OWASP LLM Top 10, AI firewall design, and SIEM integration using Python and TypeScript.

---

> ⚠️ **Disclaimer**: All projects in this repository are for **educational and research purposes only**. All attack simulations are conducted in controlled, isolated environments. No real systems, users, or data were targeted or harmed.

---

## 🚀 Research Projects

| # | Project | Security Category | Description | Stack | Status | Links |
|---|---------|-------------------|-------------|-------|--------|-------|
| 01 | **Prompt Injection Lab** | LLM01 - Prompt Injection | Proof-of-concept demonstrating direct and indirect prompt injection attacks on LLM-powered applications | Python • OpenAI API • FastAPI | ✅ Complete | [Repo](#) [Docs](#) |
| 02 | **Insecure Output Handler** | LLM02 - Insecure Output Handling | Research into unsafe LLM output handling leading to XSS and code execution vulnerabilities | TypeScript • Next.js • Claude API | ✅ Complete | [Repo](#) [Docs](#) |
| 03 | **Training Data Poisoning Sim** | LLM03 - Training Data Poisoning | Simulation of data poisoning attack vectors and their impact on model behavior | Python • Gemini API | ✅ Complete | [Repo](#) [Docs](#) |
| 04 | **Model DoS Simulator** | LLM04 - Model Denial of Service | Stress-testing LLM endpoints to simulate resource exhaustion and denial of service scenarios | Python • TypeScript • OpenAI API | ✅ Complete | [Repo](#) [Docs](#) |
| 05 | **Supply Chain Vulnerability Research** | LLM05 - Supply Chain Vulnerabilities | Research into third-party LLM plugin and dependency risks in AI pipelines | Python • TypeScript | ✅ Complete | [Repo](#) [Docs](#) |
| 06 | **AI Firewall** | Defense Layer | Custom AI firewall for intercepting and filtering malicious LLM inputs before they reach the model | Python • TypeScript • Node.js | ✅ Complete | [Repo](#) [Docs](#) |
| 07 | **SIEM Integration for AI Systems** | Monitoring & Detection | Real-time log analysis and security event monitoring for AI-powered applications using SIEM | Python • ELK Stack • Node.js | ✅ Complete | [Repo](#) [Docs](#) |
| 08 | **Sensitive Info Disclosure Lab** | LLM06 - Sensitive Information Disclosure | Exploring how LLMs can inadvertently leak sensitive training data and PII | Python • OpenAI API | 🚧 In Progress | [Repo](#) [Docs](#) |
| 09 | **Insecure Plugin Design Research** | LLM07 - Insecure Plugin Design | Research into vulnerabilities introduced by poorly designed LLM plugins and tool integrations | TypeScript • Claude API | 🚧 In Progress | [Repo](#) [Docs](#) |
| 10 | **Integrated AI Security Pipeline** | Full Stack Defense | End-to-end AI security system combining firewall, SIEM monitoring, and OWASP LLM compliance checks | Python • TypeScript • OpenAI API • ELK Stack | 🚧 In Progress | [Repo](#) [Docs](#) |

---

## 🧰 Tech Stack

**Languages & Frameworks**
![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)

**AI & LLM**
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=flat-square&logo=openai&logoColor=white)
![Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=flat-square&logo=googlegemini&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-CC785C?style=flat-square&logo=anthropic&logoColor=white)

**Security & Monitoring**
![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)
![Burp Suite](https://img.shields.io/badge/Burp_Suite-FF6633?style=flat-square&logo=burpsuite&logoColor=white)
![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=flat-square&logo=wireshark&logoColor=white)
![Elastic](https://img.shields.io/badge/ELK_Stack-005571?style=flat-square&logo=elastic&logoColor=white)
![OWASP](https://img.shields.io/badge/OWASP-000000?style=flat-square&logo=owasp&logoColor=white)

---

## 🗺️ OWASP LLM Top 10 Research Coverage

```
LLM01 - Prompt Injection                ✅ Complete
LLM02 - Insecure Output Handling        ✅ Complete
LLM03 - Training Data Poisoning         ✅ Complete
LLM04 - Model Denial of Service         ✅ Complete
LLM05 - Supply Chain Vulnerabilities    ✅ Complete
LLM06 - Sensitive Information Disclosure 🚧 In Progress
LLM07 - Insecure Plugin Design          🚧 In Progress
LLM08 - Excessive Agency               🚧 In Progress
LLM09 - Overreliance                   🚧 In Progress
LLM10 - Model Theft                    🚧 In Progress
```

---

## 📁 Project Structure

Each project folder contains:

```
project-name/
├── README.md          # Research overview, attack methodology, and findings
├── docs/              # Technical documentation and architecture diagrams
├── poc/               # Proof-of-concept implementation
└── defense/           # Mitigation strategies and defensive implementations
```

---

## 📫 Connect

- 🌐 Portfolio: [ayyannamulya.dev](https://ayyanna-mulya.vercel.app)
- 💼 LinkedIn: [in/noviyartimulyadi](https://www.linkedin.com/in/noviyartimulyadi)
- 📩 Email: [noviayya1121@gmail.com](noviayya1121@gmail.com)

---

> ⭐ All projects are research implementations in isolated environments. Full source code is kept private. Documentation and code samples are provided to demonstrate research capabilities.
>
> 🔒 No sensitive data, API keys, or credentials are included. All examples use mock data and sanitized configurations.