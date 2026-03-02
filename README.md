# Rapport SOC

# **Security Operation Center Project**

### Project Overview: Secure Corporate Infrastructure Lab

**Introduction**
In the current cyber security landscape, the "Defense in Depth" strategy is the gold standard for protecting organizational assets. This project involves the design, deployment, and hardening of a virtualized corporate network environment (`CYBERLAB.LOCAL`). The goal is to simulate a realistic small-to-medium enterprise (SME) infrastructure to test and validate security policies, network segmentation, and system hardening techniques.

**Project Objectives**
The primary objective is to move beyond simple connectivity and implement a "Secure by Design" architecture. The project is divided into three critical phases:

- **Phase 1: Infrastructure Deployment:** Establishing the Active Directory backbone, DNS services, and client workstations for distinct departments (IT, Finance, Marketing).
- **Phase 2: System Hardening (GPO):** Implementing granular Group Policy Objects to restrict access, enforce AppLocker rules, audit privilege escalation, and manage browser security.
- **Phase 3: Network Security (OPNsense):** Deploying a perimeter firewall to manage traffic flow, enforce proxy settings, and isolate the internal network from external threats.

**Scope of Work**
This lab demonstrates the practical application of Windows Server administration and network security principles. It focuses on the balance between **usability** for employees (allowing Marketing to browse, IT to script) and **security** for the organization (blocking unauthorized software, auditing admin actions).

**Technology Stack**

- **Hypervisor:** QEMU
- **Server OS:** Windows Server 2019(Domain Controller)
- **Client OS:** Windows 10 Enterprise
- **Network Security:** OPNsense Firewall & Proxy
- **Management:** Active Directory (AD DS), Group Policy Management (GPMC)

# **Phase 1 : Setup and connectivity**

**Primary Goal :** Establish a functional, isolated, and resilient virtualized Active Directory infrastructure that serves as the foundation for future Security Operations Center (SOC) simulation and monitoring.

## **Architectural Design Decisions :**

1. Hypervisor Selection: QEMU/KVM :QEMU/KVM was selected because it leverages Kernel-based Virtual Machine (KVM) technology to achieve Type 1 (bare-metal) performance characteristics. By transforming the Linux kernel itself into a hypervisor, it allows VMs to bypass the traditional Operating System translation layer required by Type 2 hypervisors (like VMware Workstation). This direct hardware access significantly reduces overhead, resulting in lower latency and near-native execution speeds.
2. Network Topology :The infrastructure utilizes a **Private Network** architecture. All virtual machines are connected to a centralized virtual switch (vSwitch) operating on the `10.0.0.0/24` subnet. This design creates a "Sandbox" environment—a completely isolated ecosystem where network traffic can be generated, monitored, and analyzed without interfering with the host machine or the external public internet.

## **Implementation details :**

**Machines:**

| **Machine Name** | **Role** | **OS** | **RAM** | **IP Address** |
| --- | --- | --- | --- | --- |
| **DC01** | Domain Controller | Windows Server 2019 | 4GB | 10.0.0.10 |
| **WIN10-CLIENT01** | Endpoint Client | Windows 10 Enterprise | 6GB | 10.0.0.20 |
- **Server DNS (127.0.0.1):** The Domain Controller (`DC01`) was configured with a loop-back address (`127.0.0.1`) as its primary DNS server. This configuration is a critical prerequisite for the proper functioning of Active Directory Domain Services (AD DS).
- **Client DNS (10.0.0.10):** the client points to the server to ensure all queries are routed through the DC, enabling future logging and domain resolution and is a crucial step for joining the domain

**Active Directory Deployment & Forest Creation**
Following the initial server configuration, the **Active Directory Domain Services (AD DS)** role was deployed to centralize identity management. The server was subsequently promoted to a Domain Controller (DC), establishing the root of a new forest named `securinets.local`. This process also involved the installation of the DNS Server role to ensure proper name resolution within the infrastructure.

**Client Domain Enrollment**
To bring the endpoint under centralized management, the client machine `WIN10-CLIENT01` was transitioned from a standalone Workgroup to the `securinets.local` domain. This operation established a trust relationship between the workstation and the Domain Controller, enabling the application of Group Policies and centralized user authentication.

### **Connectivity test:**

when pinging the servers machine :

![ping_from_client.png](Rapport%20SOC/ping_from_client.png)

---

Now trying to run nslookup to query the DNS record for `10.0.0.10` 

![image.png](Rapport%20SOC/image.png)

To resolve the DNS query timeouts observed when running `nslookup` against the server IP (`10.0.0.10`), a **Reverse Lookup Zone** was configured. This enables the server to map its IP address back to its fully qualified domain name (FQDN)."

**Configuration Steps:**

1. **Launch DNS Manager:** On the Server, navigate to **Server Manager > Tools > DNS**.
2. **Initiate Zone Creation:** In the console tree, expand the server node (**DC01**).
3. **Create New Zone:** Right-click **Reverse Lookup Zones** and select **New Zone...** to launch the wizard.

---

# Phase 2: Active Directory Design and Configuration

**Introduction**
With the network infrastructure and domain connectivity established in Phase 1, the project focus now shifts to the logical architecture of the Active Directory (AD) environment. A default AD installation places all objects into generic containers, which prevents the application of granular security controls. To transform `securinets.local` from a basic domain into a managed enterprise environment, we must implement a hierarchical design that supports departmental segregation and the principle of least privilege.

This phase is structured around three core architectural components:

**1 Organizational Unit (OU) Structure**
The foundation of effective AD management is a well-planned Organizational Unit structure. We will transition away from the default "Users" and "Computers" containers to a custom hierarchy . This segmentation is a prerequisite for applying specific Group Policies, ensuring that the Marketing department does not inherit the powerful permissions required by the IT department.

**2 Delegation of Control Matrix**
Security best practices dictate that "Domain Admin" privileges should be used sparingly. In this section, we define a delegation model that grants specific, limited rights to junior administrators . This establishes a "Tiered Administration" model essential for preventing privilege escalation attacks.

**3 Group Policy Objects (GPOs) Strategy**
Finally, we will define the strategy for policy enforcement. rather than creating a single monolithic policy, we will implement a layered GPO approach:

- **Baseline Policies:** Universal security settings applied to all workstations
- **Departmental Policies:** Specific configurations for functional groups
- **Security Overrides:** High-priority restrictions for sensitive assets (e.g., Server Hardening).

---

## **Organizational Unit (OU) Structure :**

**Implementation Details**
As illustrated in the project configuration, the `securinets.local` domain was reorganized into two primary root-level OUs:

1. **LAB_COMPUTERS:** This container manages all hardware assets joined to the domain.
    - **Workstations:** Contains client endpoints (e.g., Windows 10) subject to user-centric monitoring and AppLocker restrictions.
    - **Servers:** Contains infrastructure servers that require strict hardening and static IP configurations.
2. **LAB_USERS:** This container manages human identities and service accounts.
    - **Admins:** A protected container for high-privilege accounts, further split into `DomainAdmins` (Full Control) and `Technical Team` (Delegated Control). This separation is critical for the "Tiered Administration" model.
    - **Departments:** Defines the business logic of the organization, with sub-OUs for `Finance`, `IT`, and `Marketing`. This allows for departmental-level GPOs (e.g., Finance gets access to accounting software; Marketing gets access to social media).

**Hierarchy Diagram**
The implemented structure is represented as follows:

![Screenshot_win2k22_2026-01-10_16:49:54.png](Rapport%20SOC/Screenshot_win2k22_2026-01-10_164954.png)

**Strategic Benefits**

- **Granular GPO Targeting:** We can now link a "Block USB" policy specifically to `LAB_USERS > Departments > Finance` without affecting the IT department.
- **Delegation of Control:** The `Technical Team` OU allows us to grant Help Desk staff permission to reset passwords for `Departments` without giving them power over `DomainAdmins`.

## **Delegation of Control Matrix :**

### DomainAdmins :

The existing **Domain Admins** security group was selected to fulfill this role. While this group possesses elevated privileges by default, additional group nestings were configured to meet the specific requirements for "Schema Modification" and "Full Administrative Control."

![Pasted image.png](Rapport%20SOC/Pasted_image.png)

### Technical Team :

**Implementation Details:**
Unlike the Domain Admins configuration, the **Technical Team** was configured using **Delegation of Control**. This method ensures the team has administrative rights *only* within their assigned scopes (`COMPUTERS/Servers` and `USERS/Departments/IT`) and has zero access to the domain root.

**Configuration Actions:**

- **Targeted Delegation:**
    - **Scope 1 (Servers OU):** Used the Delegation of Control Wizard to grant the *Technical Team* rights to create, delete, and manage Computer objects.
    - **Scope 2 (IT OU):** Delegated rights to reset passwords and modify user properties for IT staff.
- **GPO Management:** Explicitly granted the "Manage Group Policy links" permission on the *Servers* and *IT* OUs only. This allows the team to apply policies to their specific units without altering the Default Domain Policy.
- **Restriction Enforcement:** The requirements for "No domain-level policy modifications" and "No schema changes" were met by **exclusion**. The group was explicitly *excluded* from the Domain Admins, Enterprise Admins, and Schema Admins groups, and was granted no permissions at the domain root level.

### Finance Department :

**Implementation Details:**
The **Finance Department** group was configured as a "Standard User" scope. Unlike the Technical Team, this group was granted no delegated administrative rights. Access was controlled strictly via NTFS Access Control Lists (ACLs) and Local Group membership on target servers.

**Configuration Actions:**

- **Resource Access (File Shares):**
    - Created a secured share `\\DC01\Financials`.
    - Granted the *Finance Department* group **Modify** permissions (Read/Write) via NTFS ACLs.
    - **Access Isolation:** Verified that the *Finance Department* group is explicitly absent from the ACLs of the Marketing and IT shared folders, satisfying the data restriction requirement.

### Marketing Department :

**Implementation Details:**
The **Marketing Department** group was configured as a Standard User scope. Access control was implemented primarily through NTFS Access Control Lists (ACLs) to ensure data isolation. The group relies on standard Group Policy Objects (GPOs) for security baselines rather than delegated administration.

**Configuration Actions:**

- **Resource Provisioning:**
    - Created the `\\DC01\Marketing` share to serve as the central repository for marketing drives and collaboration tools.
    - Assigned **Modify** permissions to the *Marketing Department* group via NTFS settings, ensuring members can create and edit documents.
- **Data Isolation (Restriction Enforcement):**
    - Enforced a strict "Deny by Default" posture regarding other departments. Verified that the *Marketing Department* group has no entry in the Access Control Lists for the `Finance` or `IT` shares.

### Workstation OU users :

**Implementation Details:**
The configuration for **Workstations OU Users** leverages the default Windows security model for Standard Users. By isolating client computers into a specific Organizational Unit (`COMPUTERS/Workstations`), policies were applied to enforce the Principle of Least Privilege.

**Configuration Actions:**

- **OU Structure & Scope:**
    - Migrated all client computer objects to the `COMPUTERS/Workstations` OU.
    - This ensures that policies targeting workstations do not accidentally affect Servers or Domain Controllers.
- **Access Levels (Standard User):**
    - Relied on the default membership of the *Domain Users* group, which maps to the local *Users* group on workstations.
    - This grants "Logon Locally" rights required for business operations but strictly denies write access to system directories (`C:\Windows`, `C:\Program Files`) and the HKLM registry hive.
- **Restriction Enforcement:**
    - **Software Installation:** Blocked implicitly by the lack of administrative tokens.
    - **Administrative Lockout:** Implemented a **Restricted Groups** policy on the Workstations OU to explicitly define the local *Administrators* group members. This ensures no general user can be accidentally or maliciously elevated to an admin role.

---

## Group Policy Objects (GPOs) Strategy

With the client machine successfully enrolled in the domain, centralized management was implemented using Group Policy Objects (GPOs). GPOs serve as the primary mechanism for defining and enforcing security configurations, user environments, and operational constraints across the `securinets.local` infrastructure. By leveraging the Group Policy Management Console (GPMC), administrative overhead is reduced while ensuring a consistent security posture for all endpoints.

---

### Default Domain Policy

To establish a robust security baseline, the **Default Domain Policy** was configured with stringent authentication requirements. Password policies now enforce mandatory **complexity** and a **14-character minimum length** to defend against credential-spraying attacks. To ensure regular credential rotation and mitigate brute-force risks, a **maximum password age of 45 days** and an **account lockout threshold of 5 attempts** were implemented. Additionally, the **Kerberos Policy** was hardened by restricting the **maximum ticket lifetime to 10 hours**, effectively shortening the window of opportunity for unauthorized session hijacking or ticket-reuse attacks .

![image.png](Rapport%20SOC/image%201.png)

---

### User Standard Policy

To further harden the user environment and minimize the internal attack surface, a series of **User Configuration** policies were implemented. Security is bolstered by an **automatic screen lock enforced after 10 minutes of inactivity**, protecting unattended workstations from unauthorized physical access. To prevent users from altering critical system configurations, **access to the Control Panel and PC Settings was disabled**. Organizational branding and consistency were maintained by applying a **standardized desktop background** to the OUs. Finally, **command-line tools (CMD/PowerShell) were restricted** for all non-IT personnel, a vital step in preventing the execution of manual scripts or local discovery commands during a potential security incident.

![image.png](Rapport%20SOC/image%202.png)

---

### Computer Hardening

To minimize the attack surface of the Windows 10 endpoints, the following security controls were implemented through a dedicated **Computer Hardening GPO**:

**Host-Based Firewall:** This step enables Windows Defender Firewall across all profiles (Domain, Private, and Public) and sets inbound connections to block by default to provide a robust host-based defense that prevents attackers from scanning the machine or moving laterally through the network.

**Guest Account Deactivation:** This step deactivates guest accounts on all workstations to eliminate anonymous access, ensuring an attacker cannot conduct reconnaissance or list network shares and user accounts without valid credentials.

**USB Auto-Play Restriction:** This step disables AutoPlay for all drives to block the automatic execution of malware, effectively preventing "Plug-and-Play" infections from compromised USB devices.

**Secure Channel Signing:** This step enforces the digital encryption and signing of secure channel data to protect the integrity of communication between workstations and the Domain Controller, mitigating the risk of Man-in-the-Middle (MITM) attacks.

**Restricted Admin Privileges:** This step restricts the local Administrators group to only authorized IT and Domain Admin accounts to enforce the Principle of Least Privilege (POLP), preventing standard users from disabling antivirus software or making unauthorized system changes.

![image.png](Rapport%20SOC/image%203.png)

![image.png](Rapport%20SOC/image%204.png)

---

**Application Whitelisting via AppLocker**

**Phase 1: System Stability and Default Rules**

To ensure that Windows remains operational, **Default Rules** were generated within the AppLocker configuration. These rules allow all users to run files located in `C:\Windows` and `C:\Program Files`. Since standard users do not have write permissions for these directories, they cannot place or execute malicious files within these trusted paths.

**Phase 2: Granular Access for Specific Roles**

Building upon the default rules, granular permissions were established to match user responsibilities:

- **Administrative Access (Sarah):** Sarah is granted full access to execute any application from any path, allowing her to perform comprehensive system maintenance and incident response tasks.
- **Security Auditing Tools (Bob):** Bob is specifically authorized to run high-risk tools like **Wireshark** and **Nmap** for auditing purposes. By restricting these tools to a specific user, the lab ensures that standard users cannot use them for unauthorized network sniffing or discovery.
- **Standard Users:** All other users remain restricted to the default rules, preventing the execution of unapproved third-party software or portable scripts.

![image.png](Rapport%20SOC/image%205.png)

---

### Server - Security Baseline

To transition the lab from basic connectivity to a true "SOC-ready" environment, additional layers of auditing and protocol restrictions were applied. These settings ensure that every critical action is logged and that the network uses modern, secure communication standards.

**Audit Object Access:** This step enables the tracking of successful or failed attempts to access specific files, folders, and registry keys to ensure that a detailed forensic trail (Event ID 4663) is created whenever sensitive data is touched, which is vital for identifying the "who, what, and when" during a data breach investigation.

![image.png](Rapport%20SOC/image%206.png)

**Disable Anonymous SID Enumeration:** This step prevents anonymous users from listing Security Identifiers (SIDs) for SAM accounts and shares to block "Null Session" attacks, where an unauthenticated attacker could otherwise harvest a list of valid usernames to use in a targeted brute-force or password-spraying campaign.

**Enforce NTLMv2:** This step configures the LAN Manager authentication level to send only NTLMv2 responses and refuse weaker LM and NTLMv1 protocols to protect the network from legacy credential-cracking tools that can easily intercept and bypass older, insecure authentication hashes.

**Restrict PowerShell Execution:** This step sets the PowerShell execution policy to `RemoteSigned` to ensure that only locally written or digitally signed scripts can run on the system, which significantly reduces the risk of users accidentally executing malicious scripts downloaded from the internet.

**Enable NTP Sync with DC:** This step forces all workstations to synchronize their system clocks with the Domain Controller's Fully Qualified Domain Name (FQDN) to ensure that time-sensitive Kerberos authentication remains functional and that all security logs across the network are perfectly aligned for chronological event correlation.

![image.png](Rapport%20SOC/image%207.png)

---

### IT Admin Tools Policy

This phase of GPO configuration focuses on providing the necessary tools for IT administration while ensuring that all administrative actions leave a permanent, unalterable forensic trail. This is a critical requirement for any SOC environment to detect and investigate privilege escalation or malicious script execution.

**Mapping the "IT_Tools" Network Share:** This step creates a hidden mapped drive pointing to `\\DC01\IT_Tools$` specifically for the **GG_IT_Admins** group to provide IT admins with a private, secure location for installers and administrative tools that is completely invisible to standard users like Alice.

![image.png](Rapport%20SOC/image%208.png)

**Disable Sleep / Hibernation:** This step disables standby states and sets the system hibernate timeout to zero to ensure that lab workstations remain powered on and accessible for remote management, automated scanning, and consistent log collection.

**Enforce Event Log Retention:** This step increases the maximum log size for Application, Security, and System logs to at least 128MB to prevent critical forensic data from being overwritten too quickly, ensuring that security analysts have a sufficient historical window to investigate past incidents.

![image.png](Rapport%20SOC/image%209.png)

**PowerShell Execution Policy:** This step enables script execution and sets the policy to "Allow local scripts and remote signed scripts" to ensure that while IT staff can run necessary management scripts, any scripts downloaded from external sources must be digitally signed by a trusted authority to execute.

**PowerShell Script Block Logging:** This step enables the recording of the actual code processed within PowerShell commands to provide the "Audit" capability required to track administrative actions and detect obfuscated or encrypted malicious scripts that traditional antivirus might miss.

**PowerShell Transcription:** This step enables the creation of a text-based "receipt" of every command typed into a PowerShell window to create a permanent, searchable record of all shell activity for forensic review and accountability.

![image.png](Rapport%20SOC/image%2010.png)

---

### Finance Application Control

This phase illustrates the GPO applied to the finance department 

**Creating the Shared Finance Folder:** This step involves creating a `C:\Finance` directory and configuring sharing permissions specifically for the **GG_Finance_Users** group to provide a centralized, secure location for departmental data while ensuring users outside of Finance cannot access the sensitive files.

**Restricting Registry Editing:** This step enables the policy to "Prevent access to registry editing tools" to stop users from using `regedit.exe` to manually bypass security controls or alter system configurations.

**Disabling Network Configuration Changes:** This step prohibits access to LAN connection properties and renames to ensure that Marketing users and others cannot change their IP addresses, preventing them from bypassing the security rules set on the **OPNsense** firewall.

Of course with an App locker configuration and a prohibited access to command prompt 

![image.png](Rapport%20SOC/image%2011.png)

![image.png](Rapport%20SOC/image%2012.png)

---

### Marketing Environment Policy

To maintain a secure and standardized environment for the Marketing Department, a specialized GPO was developed. This policy focuses on preventing unauthorized system modifications and enforcing a controlled web browsing experience through the deployment of custom Administrative Templates.

**Configuring the Mandatory Proxy Server:** This step enables a fixed proxy server setting within Microsoft Edge to route all web traffic through the **OPNsense** IP and port, ensuring that all internet activity is filtered and logged by the central security gateway.

![image.png](Rapport%20SOC/image%2013.png)

**Restricting Registry Editing:** This step enables the policy to "Prevent access to registry editing tools" to ensure that users like Charlie cannot use `regedit.exe` to bypass security controls, modify system settings, or escalate local privileges.

**Disabling Network Configuration Changes:** This step prohibits the renaming of LAN connections and restricts access to network properties to ensure that Marketing users cannot change their IP addresses, preventing them from bypassing the traffic filtering rules established on the **OPNsense** firewall.

**Disabling Time Zone and Language Changes:** This step restricts the selection of Windows menus and dialog languages to Administrators and IT staff to prevent unauthorized users from altering system-wide regional settings, which can cause synchronization issues with time-sensitive security logs.

**Edge Browser Template Integration:** This step involved downloading and importing the **Microsoft Edge ADMX templates** into the Policy Definitions folder to unlock advanced management capabilities for the Edge browser that are not available in the default Windows installation.

**Enforcing the Corporate Startup Page:** This step configures Microsoft Edge to automatically open a specific list of URLs upon startup to ensure that Marketing users are immediately directed to the mandatory corporate portal or research sites required for their roles.

![image.png](Rapport%20SOC/image%2014.png)

---

### IT Workstations Policy

To transform the lab into a functional Security Operations Center (SOC), a high-fidelity auditing strategy was implemented. These specific configurations are designed to strip away the anonymity of an attacker by recording exactly how privileges are used, when accounts are elevated, and how administrative groups are modified.

**Enable Process Creation with Command Line:** This step enables "Audit Process Creation" and the inclusion of command-line data to record the exact strings and scripts used to launch every program on the system, which is the single most effective method for detecting malicious "RunAs" commands or hidden privilege escalation scripts (Event ID 4688).

![image.png](Rapport%20SOC/image%2015.png)

**Audit Sensitive Privilege Use:** This step enables the auditing of "Sensitive Privilege Use" for both success and failure to generate logs (Event ID 4673) whenever a user attempts to exercise "Super User" rights, such as taking ownership of files or debugging system processes—actions commonly associated with manual hacking attempts.

**Audit Special Logons:** This step enables "Audit Special Logon" to create a distinct log entry (Event ID 4672) every time an account with administrative privileges logs into a workstation, allowing SOC analysts to easily filter for and monitor high-risk login activity.

**Security Group Management (Domain Controller Level):** This step enables "Audit Security Group Management" within the Default Domain Controllers Policy to record any changes made to Active Directory groups, providing a "smoking gun" (Event ID 4728) if an unauthorized user attempts to promote themselves into a restricted group like **GG_IT_Admins**.

**Enforce Advanced Audit Policy Overwrite:** This step enables the security option to "Force audit policy subcategory settings to override category settings" to ensure that the granular, advanced policies we have defined take precedence over older, legacy audit categories, preventing gaps in our forensic visibility.

![image.png](Rapport%20SOC/image%2016.png)

---

# **Phase 3: Network Security**

## Public Key Infrastructure (PKI): Enterprise Root CA Deployment

To establish cryptographic trust within the `securinets.local` domain, a Public Key Infrastructure (PKI) was required. The deployment of an internal Certificate Authority (CA) allows the organization to issue trusted SSL/TLS certificates for secure Active Directory communications (LDAPS) .

### **Implementation & Cryptographic Baseline**

The **Active Directory Certificate Services (AD CS)** role was installed and configured using modern cryptographic standards to ensure resilience against brute-force and collision attacks.
The following baseline parameters were applied during the CA configuration:

![ca_info.png](Rapport%20SOC/fd88fa8c-ccad-4743-92c8-5bda2cf41aaa.png)

Post-deployment, the Certification Authority snap-in (`certsrv.msc`) was reviewed. The CA service initialized successfully, and the newly generated Enterprise Root Certificate was automatically published to the Active Directory container, allowing all domain-joined machines to implicitly trust certificates issued by this server.

![certsrv_msc.png](Rapport%20SOC/certsrv_msc.png)

---

By default, Active Directory LDAP traffic is transmitted in **clear text** over port 389. This means anyone sniffing the network can steal credentials or view directory queries . By enabling LDAPS (port 636), we encrypt that traffic using the Certificate Authority we just built.

### Implementation Strategy

The implementation of LDAPS was achieved seamlessly by leveraging the newly deployed Enterprise Root CA. In an Active Directory environment, when an Enterprise CA is present, Domain Controllers automatically utilize the Auto-Enrollment mechanism to request a "Domain Controller" or "Kerberos Authentication" certificate.

Upon receiving this certificate into its Local Machine Personal store, the Active Directory Domain Services (ADDS) automatically binds the certificate to port 636, silently enabling LDAPS without requiring manual registry modifications or third-party proxy tools.

To validate the encrypted channel, connectivity testing was performed using the built-in Windows Directory tool (`ldp.exe`).

![image.png](Rapport%20SOC/image%2017.png)

![image.png](Rapport%20SOC/image%2018.png)

The connection was successfully established. The TLS handshake completed, verifying that the client machine trusted the issuing Root CA and that the directory traffic was fully encrypted using a strong cipher suite.

---

## Kerberos authentication

While LDAPS secures the "transmission," **Kerberos** is the "engine" that handles authentication

![image.png](Rapport%20SOC/image%2019.png)

This output confirms the successful operation of the Kerberos authentication protocol for the user `bob@SECURINETS.LOCAL` on the client workstation. The screenshot provides evidence of two critical security configurations:

1. **Successful Ticket Issuance:**
    - **Ticket #0** shows the **Ticket Granting Ticket (TGT)** issued for the `krbtgt/SECURINETS.LOCAL` service. This is the primary credential established during the initial user logon.
    - **Ticket #2** shows a **Ticket Granting Service (TGS)** ticket issued specifically for the `ldap/DC01.securinets.local` service. This ticket was generated when the client needed to perform a directory query against the Domain Controller.
2. **Strong Encryption Enforcement:**
The `KerbTicket Encryption Type` for all cached tickets is explicitly listed as **AES-256-CTS-HMAC-SHA1-96**. This verifies that the Active Directory infrastructure is enforcing the highest level of cryptography available and is not susceptible to downgrade attacks utilizing weak legacy protocols like RC4.

![image.png](Rapport%20SOC/image%2020.png)

The screenshot above displays the results of the `setspn -L DC01` command, which lists all service identities registered to the Domain Controller. This configuration is the "glue" that allows Kerberos authentication to function correctly across the network.

**Key Observations:**

- **Service Diversity:** The output shows registrations for multiple critical protocols, including `ldap/DC01.securinets.local` for directory services, `RestrictedKrbHost/DC01` for secure management, and `HOST/DC01` for general computer identification.
- **FQDN vs. NetBIOS:** Every service is registered with both its **Fully Qualified Domain Name (FQDN)** and its short **NetBIOS** name (e.g., `ldap/DC01.securinets.local` and `ldap/DC01`). This ensures that Kerberos tickets can be requested regardless of how the client addresses the server.
- **Security Impact:** By verifying these SPNs exist, we confirm that the environment is "Kerberos-ready." If any of these entries were missing, client authentication would automatically fall back to the legacy NTLM protocol, significantly increasing the risk of relay and "Pass-the-Hash" attacks.

---

## Network Security and Perimeter Defense

### OPNsense Deployment

To provide the virtualized infrastructure with a centralized gateway, firewall, and core networking services (DHCP/DNS). OPNsense was selected for its robust security features, including stateful packet inspection and its ability to export flow data to a future SIEM (Security Information and Event Management) system.
**Interface Configuration :**

| **Interface** | **QEMU Device** | **Logical Role** | **Assignment** |
| --- | --- | --- | --- |
| **WAN** | `vtnet0` | External Connectivity | Bridged/NAT to Host |
| **LAN** | `vtnet1` | Internal Private Network | Static: `10.0.0.1/24` |

**Core Network Services (DHCP)**
To automate the on-boarding of endpoint clients (such as `WIN10-CLIENT01`), a **DHCP Server** was enabled on the LAN interface. This ensures that any new machine added to the sandbox environment receives a valid IP configuration without manual intervention.

- **DHCP Range:** `10.0.0.100` — `10.0.0.200`
- **Gateway:** `10.0.0.1` (The OPNsense LAN IP)
- **Subnet Mask:** `255.255.255.0`

**Management and Security**
Access to the OPNsense management interface was hardened during the initial setup:

- **Protocol Enforcement:** The Web GUI was restricted to **HTTPS**. By opting for a newly generated self-signed certificate, we ensure that administrative credentials (`root`) are never transmitted across the internal network in plain text.
- **Access Control:** The firewall is configured to block all inbound traffic from the WAN by default, maintaining the "Sandbox" isolation required for security testing.

**Strategic Importance for the SOC**
By placing OPNsense at the edge of the `10.0.0.0/24` subnet, we have created a **Choke Point**. In later phases of this project, we will configure OPNsense to log all blocked traffic and lateral movement attempts, providing the raw data necessary for incident response simulations.