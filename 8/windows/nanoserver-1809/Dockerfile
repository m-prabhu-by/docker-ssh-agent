# escape=`

# The MIT License
#
#  Copyright (c) 2019-2020, Alex Earl and other Jenkins Contributors
#
#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to the following conditions:
#
#  The above copyright notice and this permission notice shall be included in
#  all copies or substantial portions of the Software.
#
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
#  THE SOFTWARE.

# If you pass in a POWERSHELL_VERSION, make sure it ends with a hyphen, leaving it empty will
# use the 'latest'
ARG POWERSHELL_VERSION=
FROM mcr.microsoft.com/powershell:${POWERSHELL_VERSION}nanoserver-1809

SHELL ["pwsh.exe", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ARG JAVA_VERSION=8u242-b08
ARG JAVA_SHA256=8288e4d0983019706db89c153d18bfce28d033f646be65c8ae1c33c6c65b943e
ARG JAVA_HOME=C:\jdk${JAVA_VERSION}

USER ContainerAdministrator

RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    $javaRoot = 'OpenJDK8U-jdk_x64_windows_hotspot_{0}' -f $env:JAVA_VERSION.Replace('-', '') ; `
    $url = $('https://github.com/AdoptOpenJDK/openjdk8-binaries/releases/download/jdk{0}/{1}.zip' -f $env:JAVA_VERSION, $javaRoot) ; `
    Write-Host "Retrieving $url..." ; `
    Invoke-WebRequest $url -OutFile 'openjdk.zip' -UseBasicParsing ; `
    if ((Get-FileHash openjdk.zip -Algorithm sha256).Hash -ne $env:JAVA_SHA256) { Write-Error 'Java SHA256 mismatch' ; exit 1} ; `
    Expand-Archive openjdk.zip -DestinationPath C:/ ; `
    Remove-Item -Path openjdk.zip

ENV ProgramFiles="C:\Program Files"
ENV WindowsPATH="C:\Windows\system32;C:\Windows"
ENV PATH="${WindowsPATH};${ProgramFiles}\PowerShell;${JAVA_HOME}\bin"

ARG OPENSSH_VERSION=v8.1.0.0p1-Beta

ARG user=jenkins
ARG JENKINS_AGENT_WORK="C:/Users/${user}/Work"
ENV JENKINS_AGENT_USER ${user}
ENV JENKINS_AGENT_WORK ${JENKINS_AGENT_WORK}

# setup SSH server
RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    $url = 'https://github.com/PowerShell/Win32-OpenSSH/releases/download/{0}/OpenSSH-Win64.zip' -f $env:OPENSSH_VERSION ; `
    Write-Host "Retrieving $url..." ; `
    Invoke-WebRequest -Uri $url -OutFile C:/openssh.zip -UseBasicParsing ; `
    Expand-Archive c:/openssh.zip 'C:/Program Files' ; `
    Remove-Item C:/openssh.zip ; `
    $env:PATH = '{0};{1}' -f $env:PATH,'C:\Program Files\OpenSSH-Win64' ; `
    & 'C:/Program Files/OpenSSH-Win64/Install-SSHd.ps1' ; `
    if(!(Test-Path 'C:\ProgramData\ssh')) { New-Item -Type Directory -Path 'C:\ProgramData\ssh' | Out-Null } ; `
    Copy-Item 'C:\Program Files\OpenSSH-Win64\sshd_config_default' 'C:\ProgramData\ssh\sshd_config' ; `
    $content = Get-Content -Path "C:\ProgramData\ssh\sshd_config" ; `
    $content | ForEach-Object { $_ -replace '#PermitRootLogin.*','PermitRootLogin no' `
                        -replace '#PasswordAuthentication.*','PasswordAuthentication no' `
                        -replace '#PermitEmptyPasswords.*','PermitEmptyPasswords no' `
                        -replace '#PubkeyAuthentication.*','PubkeyAuthentication yes' `
                        -replace '#SyslogFacility.*','SyslogFacility LOCAL0' `
                        -replace '#LogLevel.*','LogLevel INFO' `
                        -replace 'Match Group administrators','' `
                        -replace '(\s*)AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys','' `
                } | `
    Set-Content -Path "C:\ProgramData\ssh\sshd_config" ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value 'ChallengeResponseAuthentication no' ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value 'HostKeyAgent \\.\pipe\openssh-ssh-agent' ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value ('Match User {0}' -f $env:JENKINS_AGENT_USER) ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value ('       AuthorizedKeysFile C:/Users/{0}/.ssh/authorized_keys' -f $env:JENKINS_AGENT_USER) ; `
    New-Item -Path HKLM:\SOFTWARE -Name OpenSSH -Force | Out-Null ; `
    New-ItemProperty -Path HKLM:\SOFTWARE\OpenSSH -Name DefaultShell -Value 'C:\Program Files\Powershell\pwsh.exe' -PropertyType string -Force | Out-Null

COPY CreateProfile.psm1 C:/

# create user and user directory
RUN Import-Module -Force C:/CreateProfile.psm1 ; `
    New-UserWithProfile -UserName $env:JENKINS_AGENT_USER -Description 'Jenkins Agent User' ; `
    Remove-Item -Force C:/CreateProfile.psm1

VOLUME "${JENKINS_AGENT_WORK}" "C:/Users/${user}/AppData/Local/Temp"
WORKDIR "${JENKINS_AGENT_WORK}"

COPY setup-sshd.ps1 C:/ProgramData/Jenkins/setup-sshd.ps1

EXPOSE 22

ENTRYPOINT ["pwsh.exe", "-NoExit", "-Command", "& C:/ProgramData/Jenkins/setup-sshd.ps1"]