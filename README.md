# Active Directory Learning Journey

> Resource: [John Hammond YouTube Playlist](https://www.youtube.com/playlist?list=PL1H1sBF1VAKVoU6Q2u7BBGPsnkn-rajlp)

## Day 1: Inital Setup (HomeLab)

### Required ISOs for VMs

1. [Windows 10 Enterprise Evaluation LTSC - 64bit](https://www.microsoft.com/en-in/evalcenter/download-windows-10-enterprise)
2. [Windows Server 2022 - 64bit](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)

### Virtual Machine Setup

Using `VMware Workstation Pro`

VMware Directory Structure:

```
XYZ Active Directory Domain
└── Base Templates
    ├── Win10ProEnterpriseLTSC (VM)
    └── WinServer2022 (VM)
```

1. Windows 10 Enterprise Evaluation LTSC Setup
    - Normal VM setup w/ <br>
        `UEFI`, `2GB`, `60GB` `2x1 Cores`
    - Normal Windows 10 Pro installation
    - Installed VMware Tools
    - Installed Windows updates
    - Took a snapshot namely `Fresh Install`
    - Enabled `Template Mode` (from VM Settings > Options > Advanced)

2. Windows Server 2022 Setup
    - Normal VM setup w/ <br>
        `UEFI`, `Secure Boot`, `4GB`, `100GB` `2x1 Cores`
    - Normal Windows installation (Windows10-like)
    - Only CLI interface with `SConfig` as startup
    - Installed Windows updates (via SConfig)
    - VMware Tools not installed
    - Took a snapshot namely `Fresh Install`
    - Enabled `Template Mode` (from VM Settings > Options > Advanced)


## Day 2: Installing the Domain Controller and Joining the Home Lab

VMware Directory Structure:

```
XYZ Domain AD
├── Management Client (VM)
├── Workstations
│   └── WS01 (VM)
├── Servers
│   └── DC1 (VM)
└── Base Templates
    ├── Win10ProEnterprise (VM)
    └── WinServer2022 (VM)
```

### Virtual Machines

1. Management Client
    - Cloned `Win10ProEnterprise` from `Base Templates`
    - Set up in the `XYZ Domain AD/` (root) directory

2. DC1
    - Cloned `WinServer2022` from `Base Templates`
    - Set up in the `root/Servers/` directory
    - Installed VMware Tools

3. WS01
    - Cloned `Win10ProEnterprise` from `Base Templates`
    - Set up in the `root/Workstations/` diretory

### *Servers/DC1* Initial Setup

* Enable PowerShell Remote Execution

    ```ps
    Enable-PSRemoting
    ```

    > NOTE: This option is enabled by default on Windows Servers

* Change the default `hostname`

    - Use `SConfig` > select `2` to go to Computer Name
    - Enter new computer name: `DC1` (in this case)

### *Management Client* Initial Setup

> PowerShell in Administrator mode

* Enable Windows Remote Management Service

    ```ps
    Start-Service WinRM
    ```

* Add the server to the list of trusted hosts

    ```ps
    Set-Item wsman:\localhost\Client\TrustedHosts [server_address]
    ```

* Add the remote PS session

    ```ps
    New-PSSession -ComputerName [server_address> -Credential (Get-Credential]
    ```
    Enter the Credentials for the DC1 server.

* Enter the server's PS session

    ```ps
    Enter-PSSession [session_id]
    ```

* (Alternative) Can directly connect to the remote PS without adding it as a session listing

    ```ps
    Enter-PSSession -ComputerName [server_address] -Credential(Get-Credential)
    ```

### Active Directory Setup - On server machine i.e. `Servers/DC1`

* Set the IP address to `static`

    - Use `SConfig` > select `8` to go to Network Settings
    - Select Network Adapter, `1` in this case
    - `1` to set Network Adapter Address
        - `S` for static
        - Enter the statis IP: `192.168.138.111`
        - Default subnet mask
        - Default gateway: `192.168.138.2`
    
    > NOTE: Only the last octet of the IP is changed, the rest should be the same as the original one. The default gateway is the same as original one. Use `ipconfig /all` to see the original gateway.

* Set the `DNS Server` to point to the machine itself

    - Use `SConfig` > select `8` to go to Network Settings
    - Select Network Adapter, `1` in this case
    - Enter the DNS Server: `192.168.138.111`
    - No alternate DNS Server

* Install Active Directory Windows Feature

    ```ps
    Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
    ```

* Import AD PowerShell module

    ```ps
    Import-Module ADDSDeployment
    ```

* Install Active Directory Forest

    ```ps
    Install-ADDSForest
    ```
    Domain Name: `xyz.com` in this case <br>
    SafeModeAdministratorPassword: [any_password]

    > NOTE: It'll take a couple minutes. Also, the installation changes the `DNS Server` back to the loop address. So change it again.

* Change the `DNS server` (again) to point to the machine itself

    - Using `PowerShell` this time (`SConfig` can also be used)

    - Check the interface index of our IP
        ```ps
        Get-NetIPAddress -IPAddress 192.168.138.111
        ```
        ```shell
        IPAddress         : 192.168.138.111
        InterfaceIndex    : 6
        InterfaceAlias    : Ethernet0
        AddressFamily     : IPv4
        Type              : Unicast
        PrefixLength      : 24
        PrefixOrigin      : Manual
        SuffixOrigin      : Manual
        AddressState      : Preferred
        ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
        PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
        SkipAsSource      : False
        PolicyStore       : ActiveStore
        ```

    - Check the current `DNS server` for interface `6`
        ```ps
        Get-DNSClientServerAddress -InterfaceIndex 6
        ```
        ```shell
        InterfaceAlias               Interface Address ServerAddresses
                                     Index     Family
        --------------               --------- ------- ---------------
        Ethernet0                            6 IPv4    {127.0.0.1}
        Ethernet0                            6 IPv6    {::1}
        ```

    - Change the DNS server to point to the machine's IP
        ```ps
        Set-DnsClientServerAddress -InterfaceIndex 6 -ServerAddresses 192.168.138.111
        ```

> The Active Directory is now set up successfully. Take a snapshot of the VM, to avoid losing this state of the VM.

### Joining the created domain from `Workstations/WS01`

* Using PowerShell

    ```ps
    Add-Computer -DomainName xyz.com -Credential XYZ\Administrator -Force -Restart
    ```

    > Enter the password when prompted and the workstation will restart.

* (Alternative) Can use GUI i.e. Settings > Accounts > Access work or school > Connect > Join local active directory domain.

## Day 3: Automating Domain Users
