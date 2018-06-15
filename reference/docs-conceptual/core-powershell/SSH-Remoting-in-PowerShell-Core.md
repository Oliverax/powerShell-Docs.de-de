# <a name="powershell-remoting-over-ssh"></a>PowerShell-Remoting über SSH

## <a name="overview"></a>Übersicht

In der Regel wird beim PowerShell-Remoting für die Aushandlung der Verbindung und den Datentransport WinRM verwendet.
Für diese Remoting-Implementierung wurde Secure Shell (SSH) ausgewählt, da dieses Netzwerkprotokoll jetzt sowohl für Linux- als auch für Windows-Plattformen verfügbar ist und das PowerShell-Remoting für mehrere Plattformen ermöglicht.
Allerdings stellt WinRM auch einen stabiles Hostingmodell für PowerShell-Remotesitzungen zur Verfügung. Dies ist bei dieser Implementierung noch nicht der Fall.
Das bedeutet, dass die Remotekonfiguration von Endpunkten und „Just Enough Administration“ (JEA, Minimale Administration) für PowerShell in dieser Implementierung noch nicht unterstützt wird.

Mithilfe des SSH-Remotings für PowerShell können Sie grundlegendes PowerShell-Remoting von Sitzungen zwischen Windows- und Linux-Computern ausführen.
Dies geschieht durch Erstellen eines Hostingprozesses für PowerShell als SSH-Subsystem auf dem Zielcomputer.
Dies soll zukünftig in ein allgemeineres Hostingmodell geändert werden, das ähnlich wie WinRM funktioniert, um so Endpunktkonfigurationen und JEA zu unterstützen.

Die neuen Cmdlets „New-PSSession“, „Enter-PSSession“ und „Invoke-Command“ verfügen nun über Parameter, die diese neue Remotingverbindung vereinfachen sollen.

```powershell
[-HostName <string>]  [-UserName <string>]  [-KeyFilePath <string>]
```

Es werden voraussichtlich noch Änderungen an diesen neuen Parametern vorgenommen. Derzeit dienen sie jedoch dazu, Ihnen das Erstellen von SSH PSSessions zu ermöglichen, die Sie an die Befehlszeile koppeln oder über die Sie Befehle und Skripts aufrufen können.
Geben Sie den Zielcomputer über den HostName-Parameter an, und fügen Sie über „UserName“ einen Benutzernamen hinzu.
Wenn Sie die Cmdlets über die PowerShell-Befehlszeile ausführen, werden Sie dazu aufgefordert, ein Kennwort einzugeben.
Außerdem haben Sie die Möglichkeit, die SSH-Schlüsselauthentifizierung zu verwenden und einen Dateipfad mit einem privaten Schlüssel über den KeyFilePath-Parameter bereitzustellen.

## <a name="general-setup-information"></a>Allgemeine Setupinformationen

SSH muss auf allen Computern installiert sein.
Sie sollten sowohl den Client („ssh.exe“) als auch den Server („sshd.exe“) installieren, damit Sie das Remoting für Computer ausgiebig testen können.
Für Windows müssen Sie [Win32 OpenSSH über GitHub](https://github.com/PowerShell/Win32-OpenSSH/releases) installieren.
Für Linux müssen Sie SSH (einschließlich SSHD-Server) entsprechend Ihrer jeweiligen Plattform installieren.
Außerdem benötigen Sie einen aktuellen PowerShell-Build oder ein PowerShell-Paket von GitHub, das die Remotefunktion für SSH anbietet.
SSH-Subsysteme werden verwendet, um PowerShell-Prozesse auf dem Remotecomputer zu verwenden. Dafür muss der SSH-Server konfiguriert sein.
Zusätzlich müssen Sie die Kennwortauthentifizierung und ggf. die schlüsselbasierte Authentifizierung aktivieren.

## <a name="setup-on-windows-machine"></a>Setup auf dem Windows-Computer

1. Installieren Sie die neuste Version von [PowerShell Core für Windows]
    - Anhand der Parameter für „New-PSSession“ können Sie bestimmen, ob der Support für SSH-Remoting eingerichtet ist.

    ```powershell
    PS> Get-Command New-PSSession -syntax
    New-PSSession [-HostName] <string[]> [-Name <string[]>] [-UserName <string>] [-KeyFilePath <string>] [-SSHTransport] [<CommonParameters>]
    ```

1. Installieren Sie den neusten [Win32 OpenSSH]-Build über GitHub, und verwenden Sie dabei die [Installation].
1. Bearbeiten Sie die Datei „sshd_config“ am selben Ort, an dem Sie Win32 OpenSSH installiert haben.
    - Vergewissern Sie sich, dass die Kennwortauthentifizierung aktiviert ist.

    ```
    PasswordAuthentication yes
    ```

    - Fügen Sie einen Eintrag für das PowerShell-Subsystem hinzu, und ersetzen Sie `c:/program files/powershell/6.0.0/pwsh.exe` durch den Pfad zu der Version, die verwendet werden soll.

    ```
    Subsystem    powershell c:/program files/powershell/6.0.0/pwsh.exe -sshs -NoLogo -NoProfile
    ```

    **!!! Aktuell gibt es ein Problem in OpenSSH, das verhindert das dies funktioniert:**
    
    Siehe [subsystem executable paths with "spaces" do not work](https://github.com/PowerShell/Win32-OpenSSH/issues/784)
    
    Ein Lösungsweg ist, einen Symlink zum Powershell Installationsverzeichnis zu erstellen, der keine Leerzeichen enthält:
    
    ```powershell
    PS> mklink /D c:\pwsh "C:\Program Files\PowerShell\6.0.0"
    ```

    und dann diesen im Subsystem einzutragen:
 
    ```
    Subsystem    powershell c:\pwsh\pwsh.exe -sshs -NoLogo -NoProfile
    ```
    
    - Aktivieren Sie ggf. die Schlüsselauthentifizierung.

    ```
    PubkeyAuthentication yes
    ```

1. Starten Sie den SSHD-Dienst neu.

    ```powershell
    Restart-Service sshd
    ```

1. Fügen Sie den Pfad an der Stelle hinzu, an der OpenSSH für die Variable „Path Env“ installiert ist.
    - Dieser Pfad sollte wie folgt lauten: `C:\Program Files\OpenSSH\`.
    - So kann die Datei „ssh.exe“ gefunden werden.

## <a name="setup-on-linux-ubuntu-1404-machine"></a>Setup auf einem Linux-Computer (Ubuntu 14.04)

1. Installieren Sie den neusten Build von [PowerShell Core für Linux] über GitHub.
1. Installieren Sie ggf. [Ubuntu SSH].

    ```bash
    sudo apt install openssh-client
    sudo apt install openssh-server
    ```

1. Bearbeiten Sie die Datei „sshd_config“ unter „/etc/ssh“.
    - Vergewissern Sie sich, dass die Kennwortauthentifizierung aktiviert ist.

    ```
    PasswordAuthentication yes
    ```

    - Fügen Sie einen Eintrag für das PowerShell-Subsystem hinzu.

    ```
    Subsystem powershell /usr/bin/pwsh -sshs -NoLogo -NoProfile
    ```

    - Aktivieren Sie ggf. die Schlüsselauthentifizierung.

    ```
    PubkeyAuthentication yes
    ```

1. Starten Sie den SSHD-Dienst neu.

    ```bash
    sudo service sshd restart
    ```

## <a name="setup-on-macos-machine"></a>Setup auf einem macOS-Computer

1. Installieren Sie den neusten Build von [PowerShell Core für macOS].
    - Vergewissern Sie sich, dass SSH-Remoting aktiviert ist, indem Sie die folgenden Schritte ausführen:
      - Öffnen Sie `System Preferences`.
      - Klicken Sie auf `Sharing`.
      - Überprüfen Sie, ob für `Remote Login` `Remote Login: On` angezeigt wird.
      - Erteilen Sie den entsprechenden Benutzern Zugriff.
1. Bearbeiten Sie die Datei `sshd_config` unter `/private/etc/ssh/sshd_config`.
    - Verwenden Sie den von Ihnen bevorzugten Editor, oder

    ```bash
    sudo nano /private/etc/ssh/sshd_config
    ```

    - Vergewissern Sie sich, dass die Kennwortauthentifizierung aktiviert ist.

    ```
    PasswordAuthentication yes
    ```

    - Fügen Sie einen Eintrag für das PowerShell-Subsystem hinzu.

    ```
    Subsystem powershell /usr/local/bin/pwsh -sshs -NoLogo -NoProfile
    ```

    - Aktivieren Sie ggf. die Schlüsselauthentifizierung.

    ```
    PubkeyAuthentication yes
    ```

1. Starten Sie den SSHD-Dienst neu.

    ```bash
    sudo launchctl stop com.openssh.sshd
    sudo launchctl start com.openssh.sshd
    ```

## <a name="powershell-remoting-example"></a>Beispiel für das PowerShell-Remoting

Sie können das Remoting am einfachsten auf einem einzelnen Computer testen.
In diesem Beispiel wird eine Remotesitzung zu demselben Computer auf einem Linux-Feld erstellt.
An dieser Stelle werden PowerShell-Cmdlets über eine Eingabeaufforderung verwendet, sodass SSH-Fenster angezeigt werden, in denen Sie dazu aufgefordert werden, den Hostcomputer zu überprüfen und Kennwörter einzugeben.
Sie können auf einem Windows-Computer genauso vorgehen, um zu gewährleisten, dass das Remoting funktioniert. Stellen Sie dann eine Remoteverbindung zwischen verschiedenen Computern her, indem Sie nur den Hostnamen ändern.

```powershell
#
# Linux to Linux
#
PS /home/TestUser> $session = New-PSSession -HostName UbuntuVM1 -UserName TestUser
The authenticity of host 'UbuntuVM1 (9.129.17.107)' cannot be established.
ECDSA key fingerprint is SHA256:2kCbnhT2dUE6WCGgVJ8Hyfu1z2wE4lifaJXLO7QJy0Y.
Are you sure you want to continue connecting (yes/no)?
TestUser@UbuntuVM1s password:

PS /home/TestUser> $session

 Id Name            ComputerName    ComputerType    State         ConfigurationName     Availability
 -- ----            ------------    ------------    -----         -----------------     ------------
  1 SSH1            UbuntuVM1       RemoteMachine   Opened        DefaultShell             Available

PS /home/TestUser> Enter-PSSession $session

[UbuntuVM1]: PS /home/TestUser> uname -a
Linux TestUser-UbuntuVM1 4.2.0-42-generic 49~14.04.1-Ubuntu SMP Wed Jun 29 20:22:11 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

[UbuntuVM1]: PS /home/TestUser> Exit-PSSession

PS /home/TestUser> Invoke-Command $session -ScriptBlock { Get-Process powershell }

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName                    PSComputerName
-------  ------    -----      -----     ------     --  -- -----------                    --------------
      0       0        0         19       3.23  10635 635 powershell                     UbuntuVM1
      0       0        0         21       4.92  11033 017 powershell                     UbuntuVM1
      0       0        0         20       3.07  11076 076 powershell                     UbuntuVM1


#
# Linux to Windows
#
PS /home/TestUser> Enter-PSSession -HostName WinVM1 -UserName PTestName
PTestName@WinVM1s password:

[WinVM1]: PS C:\Users\PTestName\Documents> cmd /c ver

Microsoft Windows [Version 10.0.10586]

[WinVM1]: PS C:\Users\PTestName\Documents>

#
# Windows to Windows
#
C:\Users\PSUser\Documents>pwsh.exe
PowerShell
Copyright (c) Microsoft Corporation. All rights reserved.

PS C:\Users\PSUser\Documents> $session = New-PSSession -HostName WinVM2 -UserName PSRemoteUser
The authenticity of host 'WinVM2 (10.13.37.3)' can't be established.
ECDSA key fingerprint is SHA256:kSU6slAROyQVMEynVIXAdxSiZpwDBigpAF/TXjjWjmw.
Are you sure you want to continue connecting (yes/no)?
Warning: Permanently added 'WinVM2,10.13.37.3' (ECDSA) to the list of known hosts.
PSRemoteUser@WinVM2's password:
PS C:\Users\PSUser\Documents> $session

 Id Name            ComputerName    ComputerType    State         ConfigurationName     Availability
 -- ----            ------------    ------------    -----         -----------------     ------------
  1 SSH1            WinVM2          RemoteMachine   Opened        DefaultShell             Available


PS C:\Users\PSUser\Documents> Enter-PSSession -Session $session
[WinVM2]: PS C:\Users\PSRemoteUser\Documents> $PSVersionTable

Name                           Value
----                           -----
PSEdition                      Core
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
SerializationVersion           1.1.0.1
BuildVersion                   3.0.0.0
CLRVersion
PSVersion                      6.0.0-alpha
WSManStackVersion              3.0
PSRemotingProtocolVersion      2.3
GitCommitId                    v6.0.0-alpha.17


[WinVM2]: PS C:\Users\PSRemoteUser\Documents>
```

### <a name="known-issues"></a>Bekannte Probleme

1. Der Befehl „sudo“ funktioniert bei Remotesitzungen auf Linux-Computern nicht.

[PowerShell Core für Windows]: ../setup/installing-powershell-core-on-windows.md#msi
[PowerShell Core für Linux]: ../setup/installing-powershell-core-on-linux.md#ubuntu-1404
[PowerShell Core für macOS]: ../setup/installing-powershell-core-on-macos.md
[Win32 OpenSSH]: https://github.com/PowerShell/Win32-OpenSSH/releases
[Installation]: https://github.com/PowerShell/Win32-OpenSSH/wiki/Install-Win32-OpenSSH
[Ubuntu SSH]: https://help.ubuntu.com/lts/serverguide/openssh-server.html
