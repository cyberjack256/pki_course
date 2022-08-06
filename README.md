# pki_course
AD Basics, PKI,  Smart Card login, Third-Party HSM Lab

## Course Syllabus

### Section 1: Introduction

1. Course Introduction
2. Course Curriculum
3. Lab Requirements
4. What is Identity and Access Management?
```text
Many frameworks provide a definition as well as ways to organize, implement, and manage identity access management. Control Objectives for Information and Related technologies (COBIT), the US National Institute of Standards and Technology (NIST) Cyber Security Framework, and the International Organization for Standardization (ISO) 27000 all provide frameworks for IAM.
```

5. The Five A's of Enterprise IAM 

```Text
In their book "Identity Attack Vectors," authors Morey J. Haber and Darran Rolls provide a stripped-down definition and scope for how to see identity management as a set of universal principles that apply to all of the established security frameworks--providing a universal approach to IAM. For the purpose of this course I have adopted this universal set of principles.

The Five A's cover Authentication, Authorization, Administration, Audit, and Analytics.
```

6. Identity and Access Management Solutions?

#Active Directory Domain Controller
## Section 2: Project 1: Setup First Identity Solution for Jack's Chiropractic Center

1. Problem: Jack's employees all use credentials that are local on the endpoint they are logging into.
2. Introduction to Windows Active Directory
3. Active Directory Components (Domain, Forest, Trees, Schema and GC)
4. Create First VM in Azure for Domain Controller
5.  Setup First Active Directory
6.  DNS for Active Directory
7.  Create User and Join Computer
8.  Add a new disk to Azure VM for file share
9.  Type of Authentication in Active Directory - NTLM
10. Kerberos
11. Conclusion - Close Project 1


## PKI - Public Key Infrastructure
### Section 3: Project 2: Implement Smart Card login for Jack's Chiropractic Center using Microsoft PKI

1. Problem: Solution for Smart Card login using AD CS Microsoft PKI
2. Project Details

### Section 4: Learn Basics of Cryptography

1. What is Encryption
2. What is Digital Data singing
3. Certificates and What are Certificate Authorities


### Section 5: What are PKI and Solutions

1.  What are PKI and Solutions
3.  Introduction to Microsoft ADCS
4.  Certificate Authority and Hierarchy
5.  CAPolicy.inf file in CA



### Section 6: Deploy CA Policy CA, and SUB CA

1. Lab Infrastructure
2. Setup Root CA
3. Post Configuration of Root CA - Setup AIA & CDP
4. Setup a Webserver for AIA and CDP - Publish CRL
5. Setup GPO for Root Certificate distribution to Trusted Certificate folder
6. Setup Policy CA
7. Setup Issuing Certificate Authority
 

### Section 7: Certificate Enrollment and Template

1.  Introduction to Certificate Enrollment
2.  How Automatic Enrollment works
3.  GPO for Automatic Enrollment
4.  Certificate Template
5.  On-behalf Enrollment
6.  Web Enrollment
 

### Section 8: Smart Card based Logon for Windows - PIV

1.  Introduction to Smart Card
2.  Certificate Template for Smart Card Enrollment
3.  Smart Card Login - How it Works PIV
4.  Import User Certificate to the Smart Card and Demo User Login to Windows

### Section 9: Setup OCSP

1. What is OCSP
2. Configure OCSP in AD


### Section 10: What is HSM - Yubikey HSM

1. What is an HSM

2. HSM Configuration Demo - Yubikey HSM and AD CS
