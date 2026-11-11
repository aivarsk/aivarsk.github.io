---
layout: page
title: Resume
---

## FinTech Developer | Software Architect

**Email:** aivars.kalvans@gmail.com |
**Location:** Jelgava, Latvia |
**Web:** https://aivarsk.com |
**LinkedIn:** https://www.linkedin.com/in/aivarsk |
**GitHub:** https://github.com/aivarsk

---

Payment systems are unforgiving: a rounding error is a bug, a duplicate transfer is an incident, and "eventual consistency" doesn't survive an audit.

I design and fix the systems behind them - high-performance ledgers, banking and payment platforms - and I modernize the legacy estates they run on, Oracle Tuxedo in particular, without big-bang rewrites.

Over 20+ years that has meant:

- Designing transaction processing and ledger systems for banks and fintechs
- Migrating monolithic payment and banking systems to modern stacks
- Database performance work: query plans, concurrency, squeezing the last drop out of the engine
- Incident response and root cause analysis: production debugging, Linux observability, post-mortems that actually find the cause
- Compliance engineering: EMVCo, 3-D Secure, PCI-DSS

I wrote "Modernizing Oracle Tuxedo Applications with Python" (Packt) and speak internationally on database concurrency, performance, Python, and Linux observability - most recently at SREcon26 EMEA on hidden process state in procfs. I publish deep-dives at aivarsk.com, lately on building payment systems on TigerBeetle.

---

## Skills

- **Languages:** Python, SQL, C++, C, Java, Go
- **Frameworks:** Django, Spring Boot
- **Tech:** AWS, Docker, Kubernetes, Kafka, Oracle Database, Oracle Tuxedo, PostgreSQL, MySQL, Redis, MongoDB
- **Other:** Database performance analysis and tuning both on the database and application level, application performance, bug analysis and troubleshooting, and incident investigation.

---

## Employment history

### Senior Engineer
**PandaDoc** | *Nov 2025 - Present*

<!-- TODO: add PandaDoc bullets -->

### Owner
**Teogro OÜ** | *Apr 2022 - Present*

Independent consulting and training practice. I help banks, fintechs, and enterprises with:

- Payment and ledger system design: transaction processing, idempotency, correctness under failure
- Oracle Tuxedo modernization: incremental migration to modern stacks without big-bang rewrites
- Incident response and root cause analysis: production debugging, Linux observability, post-mortems that actually find the cause
- Database performance: query plan analysis, concurrency

### Staff Engineer
**Ebury** | *Aug 2022 - Nov 2025*

- Developed a unified interface to card issuing providers for virtual and physical cards, hiding the API differences behind a generic API and providing a unified and secure way to handle PCI DSS sensitive data.
- Migrated SWIFT messaging from MT to ISO 20022 using Python, Kubernetes, and Kafka, with a gradual switchover, and developed a control-and-repair engine that takes business rule input from spreadsheets and performs validations and data transformation for banks, countries, and currencies.
- Developed a new orchestrator service for outgoing payments that selects the best route based on rules, destination account, currency, payment amount, and time of day. Processes SEPA, Faster Payments, and SWIFT (both MT and ISO) payments through it, including ACK/NACK, confirmation, recall, and return flows.

### Senior Engineer 
**Ebury** | *Jan 2021 - Aug 2022*

- Built several query services serving data from read-optimized data structures for much faster API responses, synced from the main monolith via Debezium CDC events through Kafka and running in Kubernetes.
- Execution date calculation for payroll system taking into account SWIFT intermediaries and bank holidays in each country.

### Various positions
**Tieto** | *Feb 2002 - Dec 2020*

- Position changes: Junior Programmer, Programmer, System Analyst, Senior System Analyst, Software Architect, Lead Software Architect.
- Company name changes: Tieto Konts, TietoEnator, Tieto, TietoEVRY.
- Conducted training and developed guidelines on C++, SQL, Python, Databases, Tuxedo, and secure coding.
- Designed and developed 3DS Server and Access Control Server according to 3DSecure version 2 and got it certified.
- Developed Host Card Emulation according to MasterCard specifications and got it certified.
- Developed an accounting system, including fee and interest calculation handling thousands of transactions per second.
- Designed and developed parts of the new card issuing and authorization system.
- Worked on adopting existing software for Active-Active setup.
- Developed tools and mechanisms to make existing software PCI DSS and PA-DSS compliant with encryption schemes, key rotation, etc.
- Developed DB Stream, a database access library for C++ and Oracle on top of OCI with Tuxedo FML32, XA transactions, and bulk operations by default. It became the de-facto standard and replaced all uses of PRO*C.
- Developed and maintained a large number of libraries that all projects were using.
- Designed and developed a package manager, DB migration, and a declarative infrastructure tool for deploying Oracle Tuxedo applications consisting of ~1000 individual packages.
- Set up a CI/CD pipeline for a large C++ project based on a heavily modified Buildbot running on Linux, HP-UX, AIX, and Solaris.
- Redesigned and developed a payment gateway with configurable scenarios, extension points, and integrations (TCP/IP, XML, SOAP, ISO 8583). Used for bill payments, cash payments in ATMs, mobile and internet payments, P2P card payments, selling top-up codes in POS terminals, and selling prepaid cards.
- Designed and developed an Instant Payment Switch prototype for SEPA Credit Transfer (SCT) using ISO 20022 XML messages.
- Developed a mobile banking system communication over encrypted SMS with applications working on SIM cards.

### Implementation Specialist 
**Baltic Transit Bank** | *Apr 2001 - Aug 2001*

- Configuration and administration of Bankmaster and Branchpower systems.
- Developed extensions/customizations in C.

---

## Education

### Bachelor's in Computer Science 
**Riga Technical University** | *2000 - 2003*
- Bachelor's thesis about CORBA technology for heterogeneous distributed systems

---

## Certifications and courses

- MongoDB for Developers (M101P), MongoDB, 2013
- Documenting Software Architectures, Carnegie Mellon Software Engineering Institute
- Software Architecture Design and Analysis, Carnegie Mellon Software Engineering Institute
- Software Architecture Professional, Carnegie Mellon Software Engineering Institute
- Software Product Lines, Carnegie Mellon Software Engineering Institute

---

## Community involvement

- Wrote a book "Modernizing Oracle Tuxedo Applications with Python" and maintain https://pypi.org/project/tuxedo
- Speaking at conferences about Databases, Python, SRE, Security, and Payment Cards (https://aivarsk.com/speaking)
- [Fuxedo](https://github.com/fuxedo/fuxedo), an Open Source alternative to Oracle Tuxedo
- Several Open Source projects, including a reliable and mostly compatible Celery replacement
- JSON encoding speedup for Python 3.12
- Custom queryset in Model.refresh_from_db for Django 5.1
- Was active in GNOME2 and related projects around 2005, fixing various memory leaks, reducing memory usage to make GNOME usable on my home computer.

---

## Personal interests

- Computers and UNIX
- Brazilian jiu-jitsu (purple belt), grappling
- Winter swimming
- Participant of NATO Locked Shields cyber defence exercise
