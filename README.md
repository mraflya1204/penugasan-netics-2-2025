# Laporan Modul 2 OPREC NETICS 

## Instalasi Wazuh Manager
Wazuh manager akan diinstall pada VPS berbasis Ubuntu. Untuk melakukan instalasi lewat terminal di Ubuntu, gunakan `curl` untuk mengambil script instalasi Wazuh. 
```curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh```
Setelah mengambil script, ubah permission script yang telah diambil agar bisa di execute. Setelah itu execute untuk install Wazuh Manager
```chmod +x wazuh-install.sh && ./wazuh-install.sh```

![](media/WazuhManagerInstall1.png)

![](media/WazuhManagerInstall2.png)

After installing, we can check the dashboard for Wazuh Manager. To get into the dashboard, access the ip of where the Wazuh Server is installed. In this case, the dashboard can be accessed using https://https://46.202.164.2.

![](media/WazuhLanding.png)

Enter credentials that was given during installation, and then the dashboard will be revealed

![](media/WazuhDashboard.png)

## Instalasi Wazuh Agent
Wazuh agent akan diinstall pada sebuah Virtual Machine berbasis Linux Mint yang dijalankan pada mesin Windows 11 melewati aplikasi Virtual Box.

Untuk menginstal Wazuh Agent, pertama pindah ke root user, lalu jalankan script yang disediakan oleh Wazuh untuk melakukan install melalui `apt`

![](media/WazuhAgentInstall.png)

Setelah instalasi selesai dilakukan, Wazuh Agent akan otomatis tersambung pada Wazuh Manager yang telah ditetapkan pada variabel `WAZUH_MANAGER` pada saat instalasi.

Untuk cek apakah memang benar telah terhubung, bisa dilihat melalui Wazuh Dashboard

![](media/WazuhConnected.png)

![](media/WazuhAgentDashboard.png)

## Custom Rules
Untuk membuat custom rule, modifikasi file `local_rules.xml` yang terletak pada directory `var/ossec/etc/rules`

### Custom Rule 1 (File Permission Update)
```XML
<group name="syscheck">
  <rule id="100002" level="8">
    <if_sid>550</if_sid>
    <field name="changed_fields">permission</field>
    <description>File permission was changed.</description>
    <mitre>
      <id>T1222.002</id>
    </mitre>
  </rule>
</group>
```

Rule ini akan trigger event level 8 ketika sebuah file berubah permission.

![](media/Custom1TriggerAgent.png)

![](media/Custom1TriggerManager.png)

### Custom Rule 2 (SSH Brute Force Attempt)
```XML
<group name="ossec,syslog,sshd,">
 <rule id="100002" level="5">
   <if_sid>5716,5758,5760,5762,2502</if_sid>
   <match>^Failed|^error: PAM: Authentication|^error: maximum authentication attempts exceeded|Failed password|Failed keyboard|authentication error|Connection reset|more authentication failures;|REPEATED login failures</match>
   <description>SSH Brute Force attack is attempted </description>
   <group>authentication_failed,gdpr_IV_35.7.d,gdpr_IV_32.2,gpg13_7.1,hipaa_164.312.b,nist_800_53_AU.14,nist_800_53_AC.7,pci_dss_10.2.4,pci_dss_10.2.5,tsc_CC6.1,tsc_CC6.8,tsc_CC7.2,tsc_CC7.3,</group>
  </rule>
</group>
```

Rule ini akan trigger event level 5 ketika sebuah brute force attempt terhadap ssh terjadi.

![](media/Custom2TriggerAgent.png)

![](media/Custom2TriggerManager.png)