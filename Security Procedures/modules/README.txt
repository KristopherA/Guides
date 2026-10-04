MODULES - WORK THAT TRIAGE HANDS OFF TO
Part of: URL & Document Triage (see ../WORKFLOW.txt)
Last reviewed: 2026-09-04

WHAT THESE ARE
--------------
The four triage stages answer "is this malicious?" These modules answer the
questions that come after, and one that comes before. Each stands alone and
can be run without the triage workflow. Each is reached from a specific
handoff point, listed below.

They are deliberately not merged into the workflow. Folding a detonation lab
or a detection-engineering process into a triage procedure makes the triage
procedure unusable for triage: the reader wading through container setup to
find out whether an attachment is malicious will stop reading.

===========================================================================
detonation-lab/
===========================================================================
Containerised malware analysis: safe infrastructure, sample acquisition,
detonation, IOC extraction, YARA generation, automation, reporting.

  Reached from   stage 1, when a URL serves an executable, archive, or
                 installer rather than a document.
                 stage 2, when a file is not OOXML/OLE/PDF/RTF, or when
                 static analysis cannot resolve it.
                 stage 3, when the verdict is malicious and behaviour is
                 still needed.
  Hands back     early IOCs (domains, IPs, URLs) -> stage 1
                 unpacked payload hashes         -> stage 2 / stage 3
                 the sample's actual behaviour   -> stage 3 verdict

  00-index.txt                    entry point and the global safety rules
  01-core-workflow.txt            the eight-phase pipeline
  02-architecture.txt             lab network and container layout
  03-docker-compose.txt           the compose file
  04-pipeline-script.txt          ready-to-run automation
  05-deleted-file-monitoring.txt  watcher that recovers files the sample
                                  deletes mid-detonation
  expanded-notes.md               background: containerised analysis, the
                                  tools involved, and why not to run as root

  Read 00-index.txt before running anything in here. Detonation on a machine
  that can reach production is not analysis, it is an incident.

  Note: 01-core-workflow.txt phases 6 and 8 (detection engineering,
  reporting) overlap detection-engineering.txt. That file is the better
  version - use it and treat those phases as a pointer.

===========================================================================
macos-host-triage.txt
===========================================================================
Native-CLI sweep of a macOS workstation: running processes and connections,
persistence, binary verification, system logs, containment.

  Reached from   stage 0, when the answer to "was it opened?" is yes.
                 stage 3, whenever a document or link reached a real machine.
  Hands back     a URL from kMDItemWhereFroms  -> stage 1
                 a dropped file                -> stage 2 or the lab
                 new hashes and domains        -> detection-engineering.txt
                 confirmed persistence paths   -> detection-engineering.txt

  Merged from the former Find macOS Malware.txt and scenarios 1 and 10 of
  security-scenarios-field-guide.md, which covered the same ground twice.

===========================================================================
detection-engineering.txt
===========================================================================
The SOC process that turns analysis output into detections: intake, IOC
normalisation, detection mapping, YARA and Sigma authoring, deployment
stages, the alert triage playbook, and the feedback loop.

  Reached from   stage 3 step 10, for every malicious verdict.
                 the lab, after IOC extraction.
                 host triage, after confirming persistence.
  Hands back     new blocklist entries and network detection candidates
                 -> stage 1, as inputs to future triage
                 a fired alert -> the start of a new triage case

  Its step 7 (validation) is thin. Use detection-validation.md instead.

===========================================================================
detection-validation.md
===========================================================================
Proving a detection rule actually fires end to end, and measuring its
false-positive rate. Worked through on a Snort -> SIEM -> Sigma chain, with a
substitution table for other stacks. Offline and online staging, a stimulus
catalogue, FP baselining, coverage measurement, CI vs scheduled purple
exercises, and a validation log.

  Reached from   detection-engineering.txt, after a rule is written.
                 stage 3, when a confirmed indicator becomes a stimulus row.
  Hands back     a rule with a known FP rate - which is what makes a future
                 triage verdict trustworthy

  The most detailed module here, and the only one covering validation.
  Nothing else in this toolkit supersedes it.

===========================================================================
file-recovery.txt
===========================================================================
Recovering a deleted file from /proc/<PID>/fd while a process still holds it
open (Linux). Generic sysadmin capability rather than a security procedure.

  Reached from   the detonation lab, when a sample deletes its own payload
                 or log mid-run. See detonation-lab/05-deleted-file-
                 monitoring.txt for the automated version.
  Hands back     a recovered payload -> stage 2 or the lab

  No direct handoff from URL or document triage.

===========================================================================
security-scenarios-field-guide.md
===========================================================================
Fourteen-scenario command reference spanning endpoint malware, network
beaconing, web application attack, exposed secrets, compromised accounts,
ransomware, phishing, containers, Wi-Fi, macOS persistence, memory
forensics, vulnerability scanning, TLS, and hardening.

  Reached from   stage 3, when the case turns out to be broader than a
                 document or a link - scenario 2 for beaconing on a
                 confirmed C2 domain, scenario 5 for a user who clicked,
                 scenario 6 for ransomware behaviour, scenario 11 if
                 memory forensics is needed.

  Superseded parts, kept only because the rest of the file is intact:
    scenario 1  (endpoint malware)  -> macos-host-triage.txt
    scenario 10 (macOS persistence) -> macos-host-triage.txt
    scenario 7  (phishing email)    -> triage/, all four stages
  Use the replacements. Scenarios 2-6, 8-9, and 11-14 have no equivalent
  anywhere else here.

===========================================================================
cli-security-tools.md
===========================================================================
General security toolbox: nuclei, trivy, syft, grype, trufflehog, gitleaks,
testssl, zeek, hashcat, volatility3, lynis. Vulnerability scanning, SBOM,
secrets, TLS, PCAP, hash cracking, memory forensics, hardening.

  Reached from   nowhere in the triage workflow, with one exception:
                 testssl, for a full TLS posture audit of a host that
                 stage 1 flagged.

  Kept separate on purpose. It has essentially no overlap with URL and
  document triage, and merging it in would dilute both.

===========================================================================
HANDOFF MAP
===========================================================================

  stage 0  --was it opened?-->      macos-host-triage.txt
  stage 1  --serves an executable-> detonation-lab/
  stage 1  --TLS audit needed---->  cli-security-tools.md (testssl)
  stage 2  --unrouted container-->  detonation-lab/
  stage 2  --static not enough--->  detonation-lab/
  stage 3  --malicious verdict--->  detection-engineering.txt
  stage 3  --host involved------->  macos-host-triage.txt
  stage 3  --broader incident---->  security-scenarios-field-guide.md
  detection-engineering  --rule-->  detection-validation.md
  detonation-lab  --deleted file->  file-recovery.txt

  Everything flows back to triage/stage-3-verdict/ for the record.
