# AI Usage Log

**Platform:** Google Antigravity  
**Model:** Gemini 3.8 Flash (High)  
**Assistance Mode:** Agentic Pair Programming & Full-File Refactoring  

---

## Overview

This project was extended from Assignment 1 (Application Layer) to Assignment 2 (Transport Layer) using Google Antigravity with Gemini 3.8 Flash. The AI was tasked with analyzing the existing project architecture, designing the dual-view right panel, implementing accurate TCP/UDP transport protocol models, ensuring live synchronization across panels, and verifying TCP sequence/acknowledgment mathematics.

---

## Prompt History and Iterations

| # | Prompt (Summary) | What the AI Produced | What Was Evaluated & Corrected |
|---|------------------|----------------------|--------------------------------|
| 1 | *Assignment 1 Baseline:* Initial dual-panel dashboard creation with zero border-radius wireframe theme, supporting Browsing, Mail, and Streaming at the application layer. | Initial `index.html` featuring basic DNS, HTTP, and SMTP high-level cards with placeholder TCP handshake cards. | Ensured sharp, zero-radius monospace styling and responsive mobile layout. |
| 2 | *Assignment 2 PDF ingestion & folder structure analysis:* Ingested the assignment PDF specification. Analyzed `index.html`, `README.md`, `AI_USAGE_LOG.md`, and `REFLECTION.md`. Asked to extend the right panel to support both Application-Layer and Transport-Layer visualizations in real time while keeping the left panel unchanged. | Proposed architectural design: unified timeline engine where each step encapsulates both packet-level transport data and high-level application messages; view switcher (`Split View`, `Transport Layer`, `Application Layer`). | Approved architecture. Emphasized that Split View must display both timelines side-by-side with correlated auto-scrolling and highlighting. |
| 3 | *TCP Sequence / ACK & State Machine Implementation:* Generate accurate TCP packet streams for Browsing, Mail, and Streaming. Show exact flags (`SYN`, `SYN-ACK`, `ACK`, `PSH`, `FIN`), window size, payload length, direction, and timing. Support Relative vs. Raw 32-bit sequence numbers. | First draft of packet generation routines in `buildBrowse`, `buildMail`, and `buildStream`. | **Critical AI Correction:** The AI initially treated pure `ACK` packets as consuming 1 sequence number like `SYN`/`FIN`, and did not properly increment sequence numbers by payload length during multi-turn SMTP dialogue. Fixed sequence numbers so pure ACKs consume 0 bytes, `SYN` and `FIN` consume 1 byte, and data segments advance Seq by `len`. |
| 4 | *Streaming Transport Comparison (TCP vs. UDP):* Implement dual transport modes in Streaming to compare reliable TCP byte streams (with Slow Start cwnd expansion) against connectionless UDP datagram delivery (RTP over port 5004 with simulated packet drop). | Added transport selector dropdown in the Streaming activity panel, generating HLS-over-TCP chunks or connectionless UDP/RTP datagrams with packet drop simulation. | Verified that switching between TCP and UDP immediately resets the timeline and renders corresponding transport cards without breaking synchronization. |
| 5 | *UI Polish & Synchronization Controls:* Integrate TCP State Machine and Congestion Window status strip (`CLOSED`, `SYN_SENT`, `ESTABLISHED`, `FIN_WAIT_1`, etc.), Relative/Raw sequence toggle button (`# Rel`), speed selector, and timeline slider. | Extended controls bar and interactive state bar at top of visualizer panel. Verified that stepping backward/forward (`Prev`/`Next`) maintains strict lockstep between activity state, app cards, and transport cards. | Validated in headless browser environment; confirmed zero missing DOM IDs or runtime errors. |
| 6 | *Documentation & Reflection Paper:* Update `README.md`, `AI_USAGE_LOG.md`, and author a 1-2 page academic reflection document covering AI platform selection, synchronization mechanisms, AI errors/corrections, and protocol comparisons. | Comprehensive technical documentation and in-depth reflection paper adhering to assignment rubric. | Final review completed. All deliverables aligned with assignment requirements. |

---

## Key AI Artifacts Generated

1. `index.html`: Fully self-contained extended application with dual views, TCP state machine, and packet dissection.
2. `scratch/check_ids.py`: Automated DOM ID verification script ensuring 100% element selector alignment.
3. `README.md`: User manual and technical overview.
4. `docs/REFLECTION.md`: Detailed engineering reflection document.
