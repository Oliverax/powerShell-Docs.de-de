---
ms.date: 06/12/2017
keywords: dsc,powershell,configuration,setup
title: Optionen für Anmeldeinformationen in den Konfigurationsdaten
ms.openlocfilehash: 2c6685f3b6992537d1652f172cf926b85dd634c6
ms.sourcegitcommit: 54534635eedacf531d8d6344019dc16a50b8b441
ms.translationtype: HT
ms.contentlocale: de-DE
ms.lasthandoff: 05/16/2018
---
# <a name="credentials-options-in-configuration-data"></a>Optionen für Anmeldeinformationen in den Konfigurationsdaten
>Gilt für: Windows PowerShell 5.0

## <a name="plain-text-passwords-and-domain-users"></a>Nur-Text-Kennwörter und Domänenbenutzer

DSC-Konfigurationen mit unverschlüsselten Anmeldeinformationen generieren eine Fehlermeldung zu Nur-Text-Kennwörtern.
DSC generiert außerdem bei Verwendung von Domänenanmeldeinformationen eine Warnung.
Um diese Fehler- und Warnmeldungen zu unterdrücken, verwenden Sie die DSC-Konfigurationsdaten-Schlüsselwörter.
* **PsDscAllowPlainTextPassword**
* **PsDscAllowDomainUser**

> [!NOTE]
> Das Speichern/Übertragen von unverschlüsselten Nur-Text-Kennwörtern ist in der Regel nicht sicher. Das Sichern von Anmeldeinformationen mithilfe der weiter unten in diesem Thema behandelten Verfahren wird empfohlen.
> Der Azure Automation DSC-Dienst erlaubt Ihnen, Ihre Anmeldeinformationen zentral zu verwalten, um sie in Konfigurationen zu kompilieren und sicher zu speichern.
> Informationen finden Sie unter: [Compiling DSC Configurations / Credential Assets (Kompilieren von DSC-Konfigurationen/Anmeldeinformationsassets)](/azure/automation/automation-dsc-compile#credential-assets)

Im Folgenden finden ein Beispiel für die Weitergabe von Nur-Text-Anmeldeinformationen:

```powershell
#Prompt user for their credentials
#credentials will be unencrypted in the MOF
$promptedCreds = get-credential -Message "Please enter your credentials to generate a DSC MOF:"

# Store passwords in plaintext, in the document itself
# will also be stored in plaintext in the mof
$password = "ThisIsAPlaintextPassword" | ConvertTo-SecureString -asPlainText -Force
$username = "User1"
[PSCredential] $credential = New-Object System.Management.Automation.PSCredential($username,$password)

# DSC requires explicit confirmation before storing passwords insecurely
$ConfigurationData = @{
    AllNodes = @(
        @{
            # The "*" means "all nodes named in ConfigData" so we don't have to repeat ourselves
            NodeName="*"
            PSDscAllowPlainTextPassword = $true
        },
        #however, each node still needs to be explicitly defined for "*" to have meaning
        @{
            NodeName = "TestMachine1"
        },
        #we can also use a property to define node-specific passwords, although this is no more secure
        @{
            NodeName = "TestMachine2";
            UserName = "User2"
            LocalPassword = "ThisIsYetAnotherPlaintextPassword"
        }
        )
}
configuration unencryptedPasswordDemo
{
    Node "TestMachine1"
    {
        # We use the plaintext password to generate a new account
        User User1
        {
            UserName = $username
            Password = $credential
            Description = "local account"
            Ensure = "Present"
            Disabled = $false
            PasswordNeverExpires = $true
            PasswordChangeRequired = $false
        }
        # We use the prompted password to add this account to the local admins group
        Group addToAdmin
        {
            # Ensure the user exists before we add the user to a group
            DependsOn = "[User]User1"
            Credential = $promptedCreds
            GroupName = "Administrators"
            Ensure = "Present"
            MembersToInclude = "User1"
        }
    }

    Node "TestMachine2"
    {
        # Now we'll use a node-specific password to this machine
        $password = $Node.LocalPass | ConvertTo-SecureString -asPlainText -Force
        $username = $node.UserName
        [PSCredential] $nodeCred = New-Object System.Management.Automation.PSCredential($username,$password)

        User User2
        {
            UserName = $username
            Password = $nodeCred
            Description = "local account"
            Ensure = "Present"
            Disabled = $false
            PasswordNeverExpires = $true
            PasswordChangeRequired = $false
        }

        Group addToAdmin
        {
            Credential = $domain
            GroupName = "Administrators"
            DependsOn = "[User]User2"
            Ensure = "Present"
            MembersToInclude = "User2"
        }
    }

}
# We declared the ConfigurationData in a local variable, but we need to pass it in to our configuration function
# We need to invoke the configuration function we created to generate a MOF
unencryptedPasswordDemo -ConfigurationData $ConfigurationData
# We need to pass the MOF to the machines we named.
#-wait: doesn't use jobs so we get blocked at the prompt until the configuration is done
#-verbose: so we can see what's going on and catch any errors
#-force: for testing purposes, I run start-dscconfiguration frequently + want to make sure i'm
#        not blocked by previous configurations that are still running
Start-DscConfiguration ./unencryptedPasswordDemo -verbose -wait -force
```

## <a name="handling-credentials-in-dsc"></a>Behandeln von Anmeldeinformationen in DSC

DSC-Konfigurationsressourcen werden standardmäßig als `Local System` ausgeführt.
Einige Ressourcen benötigen jedoch Anmeldeinformationen, z. B. wenn die `Package`-Ressource Software unter einem bestimmten Benutzerkonto installieren muss.

Frühere Ressourcen verwendeten in diesem Fall einen hartcodierten `Credential`-Eigenschaftennamen.
In WMF 5.0 wurde eine automatische `PsDscRunAsCredential`-Eigenschaft für alle Ressourcen hinzugefügt.
Weitere Informationen zur Verwendung von `PsDscRunAsCredential`, finden Sie unter [Ausführen von DSC mit Benutzeranmeldeinformationen](runAsUser.md).
Neuere und benutzerdefinierte Ressourcen können diese automatische Eigenschaft verwenden, anstatt eine eigene Eigenschaften für Anmeldeinformationen zu erstellen.

> [!NOTE]
> Das Design einiger Ressourcen wurde so angelegt, dass sie aus einem bestimmten Grund mehrere Sätze von Anmeldeinformationen verwenden und über eigene Eigenschaften für Anmeldeinformationen verfügen.

Um die verfügbaren Eigenschaften für Anmeldeinformationen für eine Ressource zu finden, verwenden Sie entweder `Get-DscResource -Name ResourceName -Syntax` oder Intellisense in der ISE (`CTRL+SPACE`).

```powershell
PS C:\> Get-DscResource -Name Group -Syntax
Group [String] #ResourceName
{
    GroupName = [string]
    [Credential = [PSCredential]]
    [DependsOn = [string[]]]
    [Description = [string]]
    [Ensure = [string]{ Absent | Present }]
    [Members = [string[]]]
    [MembersToExclude = [string[]]]
    [MembersToInclude = [string[]]]
    [PsDscRunAsCredential = [PSCredential]]
}
```

In diesem Beispiel wird eine [Group](https://msdn.microsoft.com/powershell/dsc/groupresource)-Ressource aus dem integrierten DSC-Ressourcenmodul `PSDesiredStateConfiguration` verwendet.
Damit können lokale Gruppen erstellt oder Member hinzugefügt bzw. entfernt werden.
Es wird sowohl die Eigenschaft `Credential` als auch die automatische Eigenschaft `PsDscRunAsCredential` akzeptiert.
Allerdings verwendet die Ressource nur die Eigenschaft `Credential`.

Weitere Informationen über die Eigenschaft `PsDscRunAsCredential` finden Sie unter [Ausführen von DSC mit Benutzeranmeldeinformationen](runAsUser.md).

## <a name="example-the-group-resource-credential-property"></a>Beispiel: Die Eigenschaft „Credential“ der Ressource „Group“

Der DSC-Dienst wird unter `Local System` ausgeführt, sodass er bereits über Berechtigungen zum Ändern lokaler Benutzer und Gruppen verfügt.
Wenn es sich bei dem hinzugefügten Member um ein lokales Konto handelt, sind keine Anmeldeinformationen erforderlich.
Wenn die Ressource `Group` ein Domänenkonto zur lokalen Gruppe hinzufügt, sind Anmeldeinformationen erforderlich.

Anonyme Abfragen an Active Directory sind nicht zulässig.
Die Eigenschaft `Credential` der Ressource `Group`ist das zum Abfragen von Active Directory verwendete Domänenkonto.
In den meisten Fällen könnte es sich dabei um ein allgemeines Benutzerkonto handeln, da Benutzer standardmäßig über *Lesezugriff* auf die meisten Objekte in Active Directory verfügen.

## <a name="example-configuration"></a>Beispielkonfiguration

Der folgende Beispielcode verwendet DSC zum Auffüllen einer lokalen Gruppen mit einem Domänenbenutzer:

```powershell
Configuration DomainCredentialExample
{
    param
    (
        [PSCredential] $DomainCredential
    )
    Import-DscResource -ModuleName PSDesiredStateConfiguration

    node localhost
    {
        Group DomainUserToLocalGroup
        {
            GroupName        = 'ApplicationAdmins'
            MembersToInclude = 'contoso\alice'
            Credential       = $DomainCredential
        }
    }
}

$cred = Get-Credential -UserName contoso\genericuser -Message "Password please"
DomainCredentialExample -DomainCredential $cred
```

Dieser Code generiert eine Fehler- und eine Warnmeldung:

```
ConvertTo-MOFInstance : System.InvalidOperationException error processing
property 'Credential' OF TYPE 'Group': Converting and storing encrypted
passwords as plain text is not recommended. For more information on securing
credentials in MOF file, please refer to MSDN blog:
http://go.microsoft.com/fwlink/?LinkId=393729

At line:11 char:9
+   Group
At line:297 char:16
+     $aliasId = ConvertTo-MOFInstance $keywordName $canonicalizedValue
+                ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [Write-Error], InvalidOperationException
    + FullyQualifiedErrorId : FailToProcessProperty,ConvertTo-MOFInstance

WARNING: It is not recommended to use domain credential for node 'localhost'.
In order to suppress the warning, you can add a property named
'PSDscAllowDomainUser' with a value of $true to your DSC configuration data
for node 'localhost'.
```

In diesem Beispiel gibt es zwei Probleme:
1. Ein Fehler zeigt an, dass Nur-Text-Kennwörter nicht empfohlen werden.
2. Eine Warnung rät davon ab, Domänenanmeldeinformationen zu verwenden.

## <a name="psdscallowplaintextpassword"></a>PsDscAllowPlainTextPassword

Die erste Fehlermeldung enthält eine URL zur Dokumentation.
Unter diesem Link wird erläutert, wie Kennwörtern mit einer [ConfigurationData](https://msdn.microsoft.com/powershell/dsc/configdata)-Struktur und einem Zertifikat verschlüsselt werden.
Weitere Informationen zu Zertifikaten und DSC [erhalten Sie in diesen Beitrag](http://aka.ms/certs4dsc).

Um ein Nur-Text-Kennwort zu erzwingen, muss im Konfigurationsdatenabschnitt der Ressource das Schlüsselwort `PsDscAllowPlainTextPassword` wie folgt enthalten sein:

```powershell
Configuration DomainCredentialExample
{
    param
    (
        [PSCredential] $DomainCredential
    )
    Import-DscResource -ModuleName PSDesiredStateConfiguration

    node localhost
    {
        Group DomainUserToLocalGroup
        {
            GroupName        = 'ApplicationAdmins'
            MembersToInclude = 'contoso\alice'
            Credential       = $DomainCredential
        }
    }
}

$cd = @{
    AllNodes = @(
        @{
            NodeName = 'localhost'
            PSDscAllowPlainTextPassword = $true
        }
    )
}

$cred = Get-Credential -UserName contoso\genericuser -Message "Password please"
DomainCredentialExample -DomainCredential $cred -ConfigurationData $cd
```

> [!NOTE]
> `NodeName` kann kein Sternchen angegeben werden; der Name eines bestimmten Knotens ist obligatorisch.

**Microsoft empfiehlt, Nur-Text-Kennwörter aufgrund des hohen Sicherheitsrisikos zu vermeiden.**

Eine Ausnahme wäre die Verwendung des Azure Automation DSC-Diensts, da die Daten immer verschlüsselt gespeichert werden (während der Übertragung, im Dienst ruhend und ruhend auf dem Knoten).

## <a name="domain-credentials"></a>Domänenanmeldeinformationen

Wenn Sie das Beispielkonfigurationsskript erneut ausführen (mit oder ohne Verschlüsselung), wird weiterhin die Warnung generiert, dass die Verwendung von Anmeldeinformationen eines Domänenkontos nicht empfohlen wird.
Durch Verwendung eines lokalen Kontos wird verhindert, dass Domänenanmeldeinformationen, die auf anderen Servern verwendet werden könnten, offengelegt werden.

**Bei Verwendung von Anmeldeinformationen mit DSC-Ressourcen ziehen Sie, wenn möglich, ein lokales Konto einem Domänenkonto vor.**

Enthält die Eigenschaft `Username` der Anmeldeinformationen einen „\'“ oder ein „@“, behandelt DSC das Konto als Domänenkonto.
Ausnahmen machen „Localhost“, „127.0.0.1“ und „:: 1“ im Domänenteil des Benutzernamens.

## <a name="psdscallowdomainuser"></a>PSDscAllowDomainUser

Im oben stehenden Beispiel zur DSC-Ressourcen `Group` ist zum Abfragen einer Active Directory-Domäne ein Domänenkonto *erforderlich*.
Fügen Sie in diesem Fall die Eigenschaft `PSDscAllowDomainUser` wie folgt zum Block `ConfigurationData` hinzu:

```powershell
$cd = @{
    AllNodes = @(
        @{
            NodeName = 'localhost'
            PSDscAllowDomainUser = $true
            # PSDscAllowPlainTextPassword = $true
            CertificateFile = "C:\PublicKeys\server1.cer"
        }
    )
}
```

Nun generiert das Konfigurationsskript die MOF-Datei ohne Fehler oder Warnungen.