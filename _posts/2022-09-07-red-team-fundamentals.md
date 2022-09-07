---
title: THM - Red Team Fundamentals
tags: [thm, red teaming]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/5f68b8ec8a93fe7c9f5c01b0c7bbad1c.png" alt="Challenge logo" style="height:150px;">
</div>

## Introduction

Cybersecurity is a constant race between white hat hackers and black hat hackers. As threats in the cyber-world evolve, so does the need for more specialized services that allow companies to prepare for real attacks the best they can.

While conventional security engagements like vulnerability assessments and penetration tests could provide an excellent overview of the technical security posture of a company, they **might overlook some other aspects that a real attacker can exploit**. In that sense, we could say that conventional penetration tests are good at showing vulnerabilities so that you can take proactive measures but might not teach you how to respond to an actual ongoing attack by a motivated adversary.

## Vulnerability Assessment and Penetration Tests Limitations

### Vulnerability Assessments

This is the **simplest form of security assessment**, and its main objective is to **identify as many vulnerabilities in as many systems in the netwrok as possibile**. To this end, concessions may be made to meet this goal effectively. For example, the attacker's machine may be allowlisted on the available security solutions to avoid interfering with the vulnerability discovery process. This makes sense since the objective is to look at every host on the network and evaulate its security posture individually while providing the most information to the company about where to focus its remediation efforts.

To summarize, a vulnerability assessment **focuses on scanning hosts for vulnerabilities** as individual entities so that security deficiencies can be identified and effective security measures can be deployed to protect the network in a prioritized manner. Most of the work can be done with automated tools and performed by operators without requiring much technical knowledge.

If you were to run a vulnerability assessment over a network, you would normally try to scan as many of the hosts as possible, but **wouldn't actually try exploiting any vulnerabilities at all**.

### Penetration Tests

On top of scanning every single host for vulnerabilities, we often need to understand how they impact our network as a whole. Penetration tests add to vulnerability assessments by allowing the pentester to **explore the impact of an attacker** on the overall network by doing additional steps that include:

- **Attempt to exploit the vulnerabilities** found on each system. This is important as sometimes a vulnerability might exist in a system, but compensatory controls in place effectively prevent its exploitation. It also allow us to test if we can use the detected vulnerabilities to compromise a given host.
- **Conduct post-exploitation tasks** on any compromised host.

Penetration tests might start by scanning for vulnerabilities just as a regular vulnerability assessment but provide further information on how an attacker can chain vulnerabilities to achieve specific goals. While its focus remains on identifying vulnerabilities and establishing measures to protect the network, it also considers the network as a whole ecosystem and how an attacker could profit from interactions between its components.

By analyzing how an attacker could move around our network, we also gain a basic insight on possible security measure bypasses and our ability to detect a real threat actor to a certain extent, limited because the scope of a penetration test is usually extensive and Penetration testers don't care much about being loud or generating lots of alerts on security devices since time constraints on such projects often requires us to check the network in a short time.

### Advanced Persistent Threats and why Regular Pentesting is not Enough

While the conventional security engagements we have mentioned cover the finding of most technical vulnerabilities, there are limitations on such processes and the extent to which they can effectively prepare a company against a real attacker. Such limitations include:

- Time Constraints
- Budget
- Limited Scope
- Non-disruptive
- Heavy IT focus

As come consequences, some aspects of penetration tests might significantry differ from a real attack, like:

- **Penetration tests are LOUD:** Usually, pentesters won't put much effort trying to go undetected.
- **Non-technical attack vectors might be overlooked:** Attacks based on social engineering or physical intrusions are usually not included in what is tested.
- **Relaxation of security mechanisms:** While doing a regular penetration test, some security mechanisms might be temporarily disabled or relaxed for the pentesting team in favor of efficency.

Nowadays, the most prominent threat actors are known as **Advanced Persistent Threats (APT)**, which are highly groups of attackers, usually sponsored by nations or organised criminal groups. They are called persistent threat because the operations of these groups can remain undetected on compromised networks for long periods.

## Red Team Engagements

To keep up with the emerging threats, red team engagements were designed to shift the focus from regular penetration tests into a process that allows us to clearly see our defensive team's capabilities at detecting and responding to a real threat actor.

Red Team engagements consist of emulating a real threat actor's **Tactics, Techniques and Procedures (TTPs)** so that we can measure how well our blue team responds to them and ultimately improve any security controls in place.

Every red team engagement will start by defining clear goals, often referenced as crown jewels or flags, ranging from compromising a given critical host to stealing some sensitive information from the target. The red team will do everything they can to achieve the goals while remaining undetected and evading any existing security mechanisms like firewalls, antivirus, EDR, IPS and others. 

Red team engagements also improve on regular penetration tests by considering several attack surfaces:

- **Technical Infrastructure:** A red team will try to uncover technical vulnerabilities, with a much higher emphasis on stealth and evasion.
- **Social Engineering:** Targeting people through phishing campaigns, phone calls or social media to trick them into revealing information that sould be private.
- **Physical Intrusion:** Using techniques like lockpicking, RFID cloning, ecc.

Depending on the resources available, the red team exercise can be run in several ways:

- **Full Engagement:** Simulate an attacker's full workflow, from initial compromise until final goals have been achieved.
- **Assumed Breach** Start by assuming the attacker has already gained control over some assets, and try to achieve the goals from there.
- **Table-top Exercise:** An over the table simulation where scenarios are discussed between the read and the blue teams to evaluate how they would theoretically respond to certain threats.

## Engagement Structure

A core function of the red team is adversary emulation. While not mandatory, it is commonly used to assess what a real adversary would do in an environment using their tools and methodologies. The red team can use various cyber kill chains to summarize and assess the steps and procedures of an engagement.

The blue team commonly uses cyber kill chains to map behaviors and break down an adversaries movement. The red team can adapt this idea to map adversary TTPs (Tactics, Techniques, and Procedures) to components of an engagement.

Components of the kill chain are broken down in the table below.

**Technique**  | **Purpose**                      | **Examples**
---------------|----------------------------------|--------------------------
Reconnaissance | Obtain information on the target | Harvesting emails, OSINT
Weaponization  | Combine the objective with an exploit. Commonly results in a deliverable payload. | Exploit with backdoor, malicious office document
Delivery       | How will the weaponized function be delivered to the target | Emails, web, USB
Exploitation   | Exploit the target's system to execute code | MS17-010, Zero-Logon, etc.
Installation   | Install malware or other tooling | Mimikatz, Rubeus, etc.
Command & Control | Control the compromised asset from a remote central controller | Empire, Cobalt Strike, etc.
Actions on Objectives | Any end objectives: ransomware, data exfiltration, etc.	| Conti, LockBit 2.0, etc.

