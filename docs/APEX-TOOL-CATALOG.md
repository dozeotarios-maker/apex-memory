---
name: apex-tool-catalog
description: "The full researched + cited (2025-26) APEX red-team tool catalog — provisioning target list for the Parrot attack box, across enterprise + AI targets + mobile + API. Companion to apex-master-plan. Read when provisioning the VM, writing install scripts, or deciding what the agent should reach for."
metadata: 
  node_type: memory
  type: reference
  originSessionId: c87f3200-626a-43e6-9524-e9f71cedc7f6
---

# APEX Tool Catalog (researched + verified, 2025–2026)

**How to use:** this is the **provisioning target list** for the Parrot attack box, not a ceiling — the agent **installs / builds / learns** anything beyond it on demand. **OSS/CLI-first** (the autonomous agent drives these via bash); GUI/commercial tools are listed but flagged **[human-only]** (the agent can't autonomously drive them). **v1** installs the Foundational + base-glue sets; the rest is acquired per-engagement. Cross-verified against Bishop Fox, SpecterOps, NVIDIA/garak, promptfoo, Red Canary, IBM X-Force, Synacktiv, Assetnote, OWASP (2025–26).

## v1 FOUNDATIONAL (install first)
**Recon:** nmap · rustscan · masscan · subfinder · amass · httpx · naabu · katana · gau. **Web:** nuclei · ffuf · feroxbuster · gobuster · sqlmap · dalfox · arjun · seclists. **AD/Win:** netexec · bloodhound-ce+sharphound · impacket · certipy · rubeus · responder · ntlmrelayx · mitm6 · kerbrute · coercer. **Creds:** mimikatz/nanodump · pypykatz · lsassy · hashcat · john. **Privesc:** linpeas · winpeas · GodPotato. **Lateral:** ligolo-ng · chisel · proxychains · sshuttle · evil-winrm. **C2:** sliver *or* havoc. **Cloud:** pacu · ROADtools. **Report/emulate:** ghostwriter · atomic-red-team.
**Base/glue (table-stakes for a shell-driving agent):** jq · tmux/screen · nc/socat · python3+pip · curl/wget · git · build-essential · **searchsploit** (Exploit-DB local) · tcpdump/tshark · **gowitness/witnessme** (web screenshots) · script/asciinema (session recording).

## FULL CATALOG BY DOMAIN
- **Recon/OSINT:** amass · subfinder · BBOT · theHarvester · recon-ng · spiderfoot · shodan-cli · gitleaks · trufflehog · gato · cloud_enum · witnessme · gau · waybackurls · dnsx · asnmap · [human-only] Maltego.
- **Web app:** nuclei · ffuf · httpx · sqlmap · katana · dalfox · paramspider · arjun · feroxbuster · gobuster · xsstrike · jwt_tool · [human-only] Burp Suite Pro.
- **Network exploit:** nmap · rustscan · masscan · metasploit · responder · naabu · ntlmrelayx.
- **AD / Entra ID:** netexec · bloodhound-ce+sharphound · impacket (secretsdump/psexec/wmiexec/GetUserSPNs) · certipy (ADCS ESC1-16) · rubeus · mitm6 · coercer · kerbrute · GhostPack (seatbelt/sharpdpapi/certify) · powerview/sharpview · adrecon · ROADtools · AADInternals.
- **Creds + cracking:** mimikatz · nanodump · pypykatz · lsassy · donpapi · lazagne · hashcat · john · hcxdumptool/hcxtools.
- **Privesc:** PEASS-ng (lin/winpeas) · SharpUp · GodPotato/SweetPotato/MultiPotato · KrbRelayUp · GTFOBins/LOLBAS (ref) · linux-exploit-suggester · watson.
- **Lateral / pivot / tunnel:** impacket · netexec · ligolo-ng · chisel · frp · sshuttle · proxychains · evil-winrm · SharpRDP · PowerUpSQL (linked servers).
- **C2:** sliver · havoc · mythic · poshc2 · empire (BC-Security) · adaptixc2 · merlin · nimplant · [human-only/commercial] Cobalt Strike · Brute Ratel · Nighthawk.
- **Evasion / loaders / EDR-bypass:** donut · scarecrow · freeze · mangle · threatcheck · **EDRSandBlast** (BYOVD) · **EDRSilencer**/BlockEDRTraffic · **SharpBlock** · **SilentMoonwalk** (callstack spoof) · **Backstab**/unDefender · InlineExecute-Assembly · inceptor · pezor · AceLdr · syswhispers3. *(2025 frontier: sleep-masking, indirect syscalls, HW-breakpoint AMSI bypass, thread-pool injection.)*
- **Cloud:** AWS — pacu · scoutsuite · prowler · cloudmapper · enumerate-iam. Azure/M365 — ROADtools · AADInternals · GraphRunner · MicroBurst · TeamFiltration · MAAD-AF · TokenTactics. GCP — gcp_scanner · GCPBucketBrute. Multi/emulate — stratus-red-team. K8s/containers — CDK · peirates · BOtB · kube-hunter · trivy · kubescape.
- **Wireless:** aircrack-ng · hcxdumptool/tools · bettercap · wifite2 · hostapd-wpe · eaphammer.
- **Reverse engineering:** ghidra · radare2/cutter · x64dbg · frida · FLOSS · DIE · ILSpy *(dnSpy archived)* · [human-only] IDA Pro · Binary Ninja.
- **Exfiltration:** (usually via C2 channel) dnscat2 · DNSExfiltrator · SharpExfiltrate · egress-assess.
- **Persistence:** SharPersist · SharpStay · SharpHide · SharpEventPersist · reGeorg · IIS-Raid.
- **Adversary emulation:** MITRE Caldera · Atomic Red Team · Stratus Red Team · VECTR.
- **Reporting:** Ghostwriter · VECTR · PwnDoc · Dradis · RedELK · [commercial] PlexTrac.
- **C2 infra / OPSEC:** RedGuard · SourcePoint · C2concealer · GraphStrike · pwndrop · Domain Hunter · CredMaster · Terraform (infra-as-code).

## AI / LLM TARGETS (in scope)
- **LLM red-team frameworks:** **Garak** (NVIDIA — broad vuln scanner, 100+ probes; the sweep) · **PyRIT** (Microsoft — adversarial orchestration, multi-turn, agentic, XPIA; the surgical follow-up) · **promptfoo** (CI/CD LLM red-team; OWASP-LLM/NIST/ATLAS mapped; *now OpenAI-owned 2026*) · **FuzzyAI** (CyberArk — mutation/genetic jailbreak fuzzer; PAIR/ArtPrompt/ASCII-smuggling) · promptmap2 · LLM-Fuzzer (evolutionary) · **Giskard v3** · DeepTeam.
- **Adversarial ML:** **IBM ART** (evasion/poisoning/extraction/inversion across TF/PyTorch/sklearn) · TextAttack *(NLP, ~2021-era, stale)* · Counterfit *(superseded by PyRIT for LLMs)* · llm-attacks/GCG (adversarial suffixes; research-active 2025).
- **AI-infra (CVE/technique territory — no single tool):** Triton Inference Server RCE chain (2025) · exposed **MLflow**/Kubeflow endpoints · **model extraction/inversion** (ART) · **HuggingFace** supply-chain (malicious config/pickle) · **LangChain** env-var exfil CVE · **RAG poisoning** (OWASP LLM08 — invisible-text/embedding manipulation) · **MCP tool poisoning** (CVE-2025-49596, CVE-2025-6514; the new frontier — NSA CSI guidance 2025).
- **Must-haves:** Garak (sweep) + PyRIT (surgical/agentic) + promptfoo (CI regression) + FuzzyAI (mutation) + IBM ART (classical ML).

## ENTERPRISE DEPTH (the gaps generic lists miss)
- **SCCM/Intune:** **SharpSCCM** · MalSCCM · **SCCMSecrets.py** (Synacktiv) · **Maestro** (Intune/EntraID post-ex from C2, no cleartext creds). *(SCCM = Microsoft's own lateral-movement channel; consistently present, consistently overlooked.)*
- **Exchange/M365 email:** MailSniper · Ruler *(legacy on-prem)* · TeamFiltration · o365-attack-toolkit · o365recon · ntlmrelayx→EWS.
- **EDR/XDR evasion:** EDRSandBlast · Freeze · SharpBlock · SilentMoonwalk · BlockEDRTraffic · InlineExecute-Assembly · Backstab · unDefender.
- **Database:** **PowerUpSQL** · **SQLRecon** (C# successor) · mssqlproxy · NetExec-MSSQL · **ODAT** (Oracle — SID enum, TNS, priv-esc, RCE; *Oracle is in every big enterprise and gets ~zero generic coverage*).
- **Citrix/VPN/edge (CVE territory, not frameworks):** Citrix "Bleed 2" (CVE-2025-5777 / -6543 / -7775, actively exploited) · Ivanti/Pulse · GlobalProtect (CVE-2024-3400) → validate with **nuclei** CVE templates.
- **Mainframe z/OS:** **m-Ray** (IBM X-Force, 2025 — RACF/STIG scanner) · racfudit · manual RACF/APF/SVC techniques.

## MOBILE (enterprise apps)
**MobSF** (static+dynamic baseline) · **Frida + Objection** (runtime/SSL-unpin/anti-root — inseparable pair) · **jadx** (decompile to Java/Kotlin) · apktool (repackage) · **Drozer** (Android IPC/component — maintained by WithSecure, *not* deprecated) · r2frida · ghidra (ARM native libs) · [commercial] Corellium (jailbreak-free iOS), Hopper · SSL Kill Switch 3 (iOS) · Burp Mobile Assistant. *(Enterprise apps use DexGuard/iXGuard shielding → jadx + Frida are the practical path; cert-pinning bypass via Objection.)*

## API / GraphQL
- **API:** **Kiterunner** (Swagger-derived route brute-force — the differentiator vs ffuf) · **Arjun** (hidden-param discovery) · **RESTler** (Microsoft — stateful/sequence fuzzer, underrated) · ffuf · nuclei · OWASP ZAP · [human-only] Burp Pro.
- **GraphQL:** **graphw00f** (engine fingerprint) + **InQL** (Burp — schema/introspection) — treat as a pair · **Clairvoyance** (schema recon when introspection is OFF — the default hardened config; the *only* tool for it) · CrackQL · GraphQLMap · BatchQL · Graphinder · GraphCrawler. *(graphql-cop is unmaintained since 2022 — use GraphCrawler.)*

## DEPRECATED → SUCCESSOR
CrackMapExec→**NetExec** · Counterfit→**PyRIT** (LLM) · graphql-cop→**GraphCrawler** · Giskard v2→**v3** · TextAttack→**PyRIT/FuzzyAI** (LLM) · PowerSCCM→**SharpSCCM** · dnSpy→**ILSpy** · OllyDbg→**x64dbg** · Nikto→**Nuclei** (mostly) · dirb→**ffuf** · Fierce→**Amass/Subfinder**.

## NOTES
- **Commercial/GUI tools** (Cobalt Strike, Burp, IDA, Nighthawk, Corellium, PlexTrac) are listed for completeness but excluded from autonomous use — the agent uses OSS/CLI equivalents; humans use these.
- **AI-infra & Citrix/VPN & mainframe** are largely **CVE/technique** domains, not framework domains — handled by nuclei + metasploit + web research + the Code Cell, not a dedicated tool.
- Irony worth noting: **MCP tool-poisoning** is itself an AI-attack surface — relevant for AI-target engagements even though APEX's own design dropped MCP.
- Full citations live in the research transcripts; key sources: Bishop Fox 2025 RT tools, SpecterOps (SharpSCCM/BloodHound/GhostPack), NVIDIA garak, promptfoo blog, IBM X-Force (m-Ray/Intune), Synacktiv (SCCMSecrets), Assetnote (Kiterunner), OWASP LLM Top-10 2025, NSA MCP CSI.
