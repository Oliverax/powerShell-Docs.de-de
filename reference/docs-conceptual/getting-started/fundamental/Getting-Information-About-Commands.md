---
ms.date: 06/05/2017
keywords: powershell,cmdlet
title: Abrufen von Informationen über Befehle
ms.assetid: 56f8e5b4-d97c-4e59-abbe-bf13e464eb0d
ms.openlocfilehash: c51579fe2cdf4f2a0d3248d1aaf3f1f9cac83868
ms.sourcegitcommit: 01d6985ed190a222e9da1da41596f524f607a5bc
ms.translationtype: HT
ms.contentlocale: de-DE
ms.lasthandoff: 06/07/2018
ms.locfileid: "34482725"
---
# <a name="getting-information-about-commands"></a>Abrufen von Informationen über Befehle
Das Windows PowerShell-Cmdlet `Get-Command` ruft alle Befehle ab, die in Ihrer aktuellen Sitzung verfügbar sind. Wenn Sie `Get-Command` bei einer Windows PowerShell-Eingabeaufforderung eingeben, erhalten Sie eine Ausgabe wie die folgende:

```
PS> Get-Command
CommandType     Name                            Definition
-----------     ----                            ----------
Cmdlet          Add-Content                     Add-Content [-Path] <String[...
Cmdlet          Add-History                     Add-History [[-InputObject] ...
Cmdlet          Add-Member                      Add-Member [-MemberType] <PS...
...
```

Diese Ausgabe sieht der Hilfe-Ausgabe von „Cmd.exe“ sehr ähnlich: eine tabellarische Zusammenfassung der internen Befehle. Im Auszug der oben gezeigten **Get-Command**-Befehlsausgabe hat jeder aufgeführte Befehl den Befehlstyp (CommandType) „Cmdlet“. Ein Cmdlet ist der systeminterne Befehlstyp in Windows PowerShell, d.h. ein Typ, der ungefähr den Befehlen **dir** und **cd** von „Cmd.exe“ sowie integrierten Befehlen von UNIX-Shells wie BASH entspricht.

In der Ausgabe des Befehls `Get-Command` enden alle Definitionen mit Auslassungspunkten (...), um anzugeben, dass PowerShell nicht alle Inhalte im verfügbaren Platz anzeigen kann. Wenn Windows PowerShell Ausgabe anzeigt, wird diese als Text formatiert und so angeordnet, dass sie genau ins Fenster passt. Dies wird später in dem Abschnitt erläutert, in dem es um Formatierer geht.

Das Cmdlet `Get-Command` hat einen **Syntax**-Parameter, mit dem die Syntax jedes Cmdlets abgerufen werden kann. Verwenden Sie den folgenden Befehl, um die Syntax des Cmdlets „Get-Help“ abzurufen:

```
Get-Command Get-Help -Syntax

Get-Help [[-Name] <String>] [-Path <String>] [-Category <String[]>] [-Component <String[]>] [-Functionality <String[]>]
 [-Role <String[]>] [-Full] [-Online] [-Verbose] [-Debug] [-ErrorAction <ActionPreference>] [-WarningAction <ActionPreference>] [-ErrorVariable <String>] [-WarningVariable <String>] [-OutVariable <String>] [-OutBuffer <Int32>]

Get-Help [[-Name] <String>] [-Path <String>] [-Category <String[]>] [-Component <String[]>] [-Functionality <String[]>]
 [-Role <String[]>] [-Detailed] [-Online] [-Verbose] [-Debug] [-ErrorAction <ActionPreference>] [-WarningAction <ActionPreference>] [-ErrorVariable <String>] [-WarningVariable <String>] [-OutVariable <String>] [-OutBuffer <Int32>]

Get-Help [[-Name] <String>] [-Path <String>] [-Category <String[]>] [-Component <String[]>] [-Functionality <String[]>]
 [-Role <String[]>] [-Examples] [-Online] [-Verbose] [-Debug] [-ErrorAction <ActionPreference>] [-WarningAction <ActionPreference>] [-ErrorVariable <String>] [-WarningVariable <String>] [-OutVariable <String>] [-OutBuffer <Int32>]

Get-Help [[-Name] <String>] [-Path <String>] [-Category <String[]>] [-Component <String[]>] [-Functionality <String[]>]
 [-Role <String[]>] [-Parameter <String>] [-Online] [-Verbose] [-Debug] [-ErrorAction <ActionPreference>] [-WarningAction <ActionPreference>] [-ErrorVariable <String>] [-WarningVariable <String>] [-OutVariable <String>] [-OutBuffer <Int32>]
```

### <a name="displaying-available-command-types"></a>Anzeigen von verfügbaren Befehlstypen
Der Befehl **Get-Command** listet nicht jeden der Befehle auf, der in Windows PowerShell verfügbar sind. Stattdessen listet der Befehl **Get-Command** nur die Cmdlets auf, die es in der aktuellen Sitzung gibt. Windows PowerShell unterstützt tatsächlich mehrere weitere Befehlstypen. Aliase, Funktionen und Skripts sind ebenfalls Windows PowerShell-Befehle, obwohl sie nicht ausführlich im Windows PowerShell-Benutzerhandbuch erläutert werden. Auch externe Dateien, die ausführbar sind oder einen registrierten Dateityphandler haben, werden als Befehle klassifiziert.

Um alle Befehle in der Sitzung aufzurufen, geben Sie Folgendes ein:

```powershell
Get-Command *
```

Da diese Liste externe Dateien in Ihrem Suchpfad einschließt, kann sie Tausende von Elementen enthalten. Es ist sinnvoller, einen reduzierten Satz von Befehlen anzuzeigen.

Um native Befehle anderer Typen abzurufen, verwenden Sie den **CommandType**-Parameter des Cmdlets `Get-Command`.

> [!NOTE]
> Das Sternchen (\*) wird zum Abgleichen mit Platzhalterzeichen in Windows PowerShell-Befehlsargumenten verwendet. \* bedeutet „Abgleichen mit einem oder mehreren beliebigen Zeichen“. Beispielsweise können Sie `Get-Command a*` eingeben, um nach allen Befehlen zu suchen, die mit dem Buchstaben „a“ beginnen. Anders als beim Abgleichen mit Platzhalterzeichen in „Cmd.exe“ stimmt das Windows PowerShell-Platzhalterzeichen auch mit einem Punkt überein.

Geben Sie Folgendes ein, um Befehlsaliase abzurufen, bei denen es sich um die zugewiesenen Spitznamen von Befehlen handelt:

```powershell
Get-Command -CommandType Alias
```

Um die Funktionen in der aktuellen Sitzung abzurufen, geben Sie Folgendes ein:

```powershell
Get-Command -CommandType Function
```

Um Skripts im Suchpfad von Windows PowerShell anzuzeigen, geben Sie Folgendes ein:

```powershell
Get-Command -CommandType Script
```
