---
title: 'ExchangeOnline Mailadressen anzeigen, hinzufügen, entfernen'
date: 2021-05-25
permalink: /posts/2021/05/ExchangeOnline-Mailadressen/
tags:
  - ExchangeOnline
  - Mailadressen
---

ExchangeOnline Mailadressen anzeigen, hinzufügen, entfernen'

# Exchange Online: Mailadressen anzeigen, hinzufügen, entfernen
## Voraussetzungen
Das ExchangeOnlineManagment Module muss installiert, importiert sein und man muss sich mit der entsprechenden Exchange Online Umgebung verbinden: 

ExchangeOnlineManagement Module in Powershell installieren:
``Install-Module -Name ExchangeOnlineManagement``

Das Module importieren:
``Import-Module ExchangeOnlineManagement``

Mit der Exchange Online Environment verbinden (run this command as admin, then login)
``Connect-ExchangeOnline -UserPrincipalName admin@customerdomain.onmicrosoft.com``

Für die Befehle im lokalen AD, muss das ActiveDirectory Powershell Modul geladen sein: https://theitbros.com/install-and-import-powershell-active-directory-module/

## Hinweise
Die Verwaltung von Mailadressen auf Postfächern verhält sich anders in einem Setup mit einem on-prem active directory und eingerichtetem AD Connect, als wenn nur rein Exchange Online User bestehen. 

## Auslesen der Mailaddressen
Mit folgendem Befehl werden alle Mailadressen der Mailbox test-unico ausgelesen und angezeigt:

``Get-Mailbox -Identity test-unico | select EmailAddresses | fl``



## Mailadressen hinzufügen / entfernen
Es ist theoretisch auch möglich so Mailadressen zu hinzufügen, aber ACHTUNG: die folgenden Befehle funktionieren nur in einem reinen Exchange Online Szenario. 

``Set-Mailbox "Dan Jump" -EmailAddresses @{add="dan.jump@northamerica.contoso.com"}

Set-Mailbox "Dan Jump" -EmailAddresses @{add="dan.jump@northamerica.contoso.com","danj@tailspintoys.com"}``

oder zu entfernen:

``Set-Mailbox "Janet Schorr" -EmailAddresses @{remove="janets@corp.contoso.com"}``

### Szenario Onprem AD mit Synch
Ist noch ein OnPrem AD mit einem Synch vorhanden, dann müssen Mailadressen über das AD gesetzt, entfernt werden:

Die Mailadressen werden in diesem Fall dem User AD Attribut ProxyAddresses hinzugefügt oder entfernt, sobald dann der AD Synch in die Cloud durchläuft werden die Adressen dort entsprechend angepasst. 

Dem Benutzer paulie eine Mailadresse hinzufügen:

``Set-ADUser paulie -Add @{ProxyAddresses="SMTP:primaryaddress@tachytelic.net"}``

Dem Benutzer paulie eine Mailadresse entfernen:

``Set-ADUser paulie -Remove @{ProxyAddresses="smtp:paulietest@tachytelic.net"}``