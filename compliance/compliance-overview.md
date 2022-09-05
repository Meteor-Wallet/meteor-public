# Meteor Wallet Statement of Compliance

This document journals Meteor Wallet's progress towards compliance for wallets in the Near ecosystem, as detailed [here](https://github.com/near/wallet-selector/blob/main/CONTRIBUTING.md#wallet-security-criteria).

As we develop Meteor we are always acutely aware of the responsibility we have towards our users and the security of their interactions with our wallet. We believe we have taken every precaution towards this end, but we know there is always more to be done to maintain confidence and trust in the ecosystem. We pledge to do everything we can to meet the criteria set below, and a more detailed outline of our progress in meeting it will soon follow.

---

## Required Criteria elements

1. Has a security program in place that covers or is dedicated to the wallet and...

2. Publishes information about its security program in an easily findable place.

3. Conducts regular audits of wallet code, at regular intervals of less than a year or based on meaningful code changes.

4. Conducts regular penetration tests, both “authenticated” and “non-authenticated” upon significant code changes.

5. Conducts penetration tests on related infrastructure, such as databases, virtual machines, web servers, etc.

6. Remediates any critical, high, or medium findings from audits (3, 4, an 5 above) in a rapid fashion, as suggested by auditors.  Auditors should validate the remediation in their reports.

7. Makes such reports (audits, penetration tests) publicly available, on at least a summary level.

8. When making reports (7) available, wallet projects should ensure the equivalent reports appear on the security vendor’s site or simply links to the security vendor’s report. This ensures the authenticity of the audit reports.

9. Conducts operational readiness reviews and testing or an equivalent process before deployment to production to ensure that code changes have not resulted in unanticipated behavior, compatibility issues, or inclusion of vulnerabilities.

10. Maintains a testnet wallet, available to developers and security researchers.

11. Maintains a bug bounty program.

12. Implements minimal privilege and access policies with regard to supporting infrastructure.

13. Implements MFA and strong passwords for access to critical related systems, such as domain registration, hosting platforms, cloud platforms, etc.

14. Conducts known vulnerability and vulnerable dependency checks; and remediates critical, medium, and high findings before deploying to testnet or production.

15. If the wallet is a  browser “extension,” it is listed on the official extension marketplace.

16. Logs are collected from supporting infrastructure, web servers, etc.

17. Ensures that logs do not contain sensitive information, are encrypted at rest in storage, and have restricted access with least privilege.

18. Enables “audit logs” on related platforms (AWS, GCP, monitoring platforms, etc).

19. Logs shall be maintained for 90 days for forensic purposes.

20. Security feature shall be enabled in hosting environments, i.e. AWS GuardDuty.

21. Inputs shall be sanitized.

22. Code SAST scanning for vulnerabilities and vulnerable dependencies shall be conducted prior to production.

23. Vulnerability scanning shall be conducted regularly on websites, infrastructure-related VMs, etc.  Findings shall be quickly remediated.

24. Patch management shall be conducted frequently on supporting infrastructure.

25. VM and bare-metal infrastructure shall be protected by endpoint detection and response software.

26. An incident monitoring, alerting, and response program shall be in place.

27. Additional protective technologies, such as web application firewalls, should be put in place and appropriately configured to prevent attacks on the wallet.

28. White and black lists should be maintained for ip addresses and domains interacting with the wallet to the extent possible.

29. Reduction in attack surface shall be conducted by removing access to paths and unused ports; properly configuring domains, CSP, etc.

30. Access to infrastructure should be limited to VPNs or commensurate technology, IP restricted, etc. in order to prevent malicious access. For example, port 22 for the web server would not be accessible from the internet.

31. OWASP top 10 vulnerabilities shall be regularly tested and remediated.

32. Tools such as Nessus and Qualys, or similar should be used to scan for vulnerabilities.

33. Tools such as Burp Suite, or similar should be used to instrument and and analyze the interaction of the wallet with the browser for security issues.

34. Playbooks for public disclosure and communication to stakeholders and users should be prepared in advance and practiced via tabletop exercises in order to ensure rapid response and disclosure to protect such stakeholders and their assets.

35. An incident response retainer should be considered.

36. Wallet projects should clearly inform users of risks, risky behavior, best practices in the use of the wallet, and the most secure methods for using the wallet.

37. Wallet projects should have a clear path to reporting and receiving help on issues that is easily found by users.

38. Sensitive information should not be stored in the client-side wallet in a way that could be scraped from storage by malicious software, such as keys, recovery phrases, etc.

39. Encryption of sensitive data should likewise not rely on hard-coded keys or weak ciphers.

40. A user’s ability to recover their wallet under various circumstances should be clearly spelled out to the user when setting up the wallet.

41. Communication between infrastructure elements should be secured to the maximum extent possible.