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

### Rule 1 Prequisite
Testing untuk FIM akan dilakukan pada sebuah directory spesial bernama `specialdir3` yang terletak pada home. Tambahkan line berikut di Wazuh Agent pada file `/var/ossec/etc/ossec.conf`:
```XML
<ossec_config>
  <syscheck>
    <directories realtime="yes">/specialdir3</directories>
  </syscheck>
</ossec_config>
```
Hal ini dilakukan agar Wazuh Agent dapat mendeteksi kejadian yang ada pada folder tersebut. Apabila tidak ditambahkan, tidak akan ada deteksi yang dapat trip Wazuh Rules.

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

### Custom Rule 3 (Shell Script Permission Changed)
```XML
<group name="syscheck">
  <rule id="100003" level="8">
    <if_sid>550</if_sid>
    <field name="file">.sh$</field>
    <field name="changed_fields">^permission$</field>
    <description>Permission change detected on shell script.</description>
    <mitre>
      <id>T1222.002</id>
    </mitre>
  </rule>
</group>
```

Rule ini akan trigger event level 8 ketika sebuah file berekstensi `.sh` (shell script) diubah permission nya.

![](media/Custom3TriggerAgent.png)

![](media/Custom3TriggerManager.png)


### Rule 4-6 Prequisite
Untuk Rule 4, 5, dan 6. Terlebih dahulu konfigurasi file `/var/ossec/etc/ossec.conf` pada Wazuh Agent dengan menambahkan sebagai berikut:
```conf
<ossec_config>
  <syscheck>
    <directories check_all="yes" realtime="yes">/root/.ssh/</directories>
    <directories check_all="yes" realtime="yes">/home/*/.ssh/</directories>
    <directories check_all="yes" realtime="yes">/var/*/.ssh/</directories>
    <directories check_all="yes" realtime="yes">/etc/ssh/sshd_config</directories>
  </syscheck>
</ossec_config>
```
Hal ini dilakukan agar Wazuh Agent dapat memonitor file file yang berada pada semua directory ssh.

![](media/Custom4AssetOssec.png)

### Custom Rule 4 (SSH authorized_keys file is added)
```XML
<group name="common_persistence_techniques,sshd,">
  <rule id="100004" level="10">
    <if_sid>554</if_sid>
    <field name="file" type="pcre2">\/authorized_keys$</field>
    <description>SSH authorized_keys file "$(file)" has been added</description>
    <mitre>
      <id>T1098.004</id>
    </mitre>
  </rule>
```

Rule ini akan trigger event level 10 ketika sebuah authorized_keys ditambahkan pada ssh Wazuh Agent.

![](media/Custom4TriggerAgent.png)

![](media/Custom4TriggerManager.png)

### Custom Rule 5 (SSH authorized_keys file is modified)
```XML
  <rule id="100005" level="10">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">\/authorized_keys$</field>
    <description>SSH authorized_keys file "$(file)" has been modified</description>
    <mitre>
      <id>T1098.004</id>
    </mitre>
  </rule>
```

Rule ini akan trigger event level 10 ketika sebuah authorized_keys yang sudah ada dimodifikasi dalam bentuk apapun pada Wazuh Agent.

![](media/Custom5TriggerAgent.png)

![](media/Custom5TriggerManager.png)

### Custom Rule 6 (SSH config has been modified)
```XML
  <rule id="100006" level="10">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">\/sshd_config$</field>
    <description>SSH config file "$(file)" has been modified</description>
    <mitre>
      <id>T1098.004</id>
    </mitre>
  </rule>
```

Rule ini akan trigger event level 10 ketika config dari SSHD telah dimofikasi

![](media/Custom6TriggerAgent.png)

![](media/Custom6TriggerManager.png)

### Rule 7 Prequisites
Untuk rule 7, konfigurasi file `/var/ossec/etc/ossec.conf` dan tambahkan:
```XML
<ossec_config>
  <syscheck>
    <directories check_all="yes" realtime="yes">/etc/shadow</directories>
    <directories check_all="yes" realtime="yes">/etc/gshadow</directories>
    <directories check_all="yes" realtime="yes">/etc/passwd</directories>
    <directories check_all="yes" realtime="yes">/etc/group</directories>
    <directories check_all="yes" realtime="yes">/etc/login.defs</directories>
  </syscheck>
</ossec_config>
```

### Rule 7
```XML
<group name="common_persistence_techniques,">
  <rule id="100007" level="10">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">\/etc\/passwd$|\/etc\/shadow$|\/etc\/gshadow$|\/etc\/group$|\/etc\/login.defs$</field>
    <description>[File "$(file)" has been modified]: Possible local account manipulation</description>
    <mitre>
      <id>T1136.001</id>
      <id>T1078.003</id>
    </mitre>
  </rule>
</group>
```
Rule ini akan trigger event level 10 ketika beberapa folder, mainly `/etc/shadow`, `/etc/gshadow`, `/etc/passwrd`, `/etc/group`, `/etc/login.defs` telah dimodifikasi. Ketika folder-folder tersebut dimodifikasi, maka terdapat indikasi bahwa ada sebuah local account pada mesin yang telah tercompromised.

![](media/Custom7TriggerAgent.png)

![](media/Custom7TriggerManager.png)

### Rule 8 - 9 Prequisites
Untuk rule 8 - 9, konfigurasi file `/var/ossec/etc/ossec.conf` dan tambahkan:
```XML
<ossec_config>
  <syscheck>
    <directories check_all="yes" realtime="yes">/etc/</directories>
    <directories check_all="yes" realtime="yes">/home/*/.bash_profile</directories>
    <directories check_all="yes" realtime="yes">/home/*/.bash_login</directories>
    <directories check_all="yes" realtime="yes">/home/*/.profile</directories>
    <directories check_all="yes" realtime="yes">/home/*/.bash_profile</directories>
    <directories check_all="yes" realtime="yes">/home/*/.bashrc</directories>
    <directories check_all="yes" realtime="yes">/home/*/.bash_logout</directories>
    <directories check_all="yes" realtime="yes">/root/.bash_profile</directories>
    <directories check_all="yes" realtime="yes">/root/.bash_login</directories>
    <directories check_all="yes" realtime="yes">/root/.profile</directories>
    <directories check_all="yes" realtime="yes">/root/.bash_profile</directories>
    <directories check_all="yes" realtime="yes">/root/.bashrc</directories>
    <directories check_all="yes" realtime="yes">/root/.bash_logout</directories>
  </syscheck>
</ossec_config>
```

### Rule 8
```XML
<group name="common_persistence_techniques,">
  <rule id="100008" level="10">
    <if_sid>554</if_sid>
    <field name="file" type="pcre2">\/etc\/profile$|\/etc/profile.d\/|\/etc\/bash.bashrc$|\/etc\/bash.bash_logout$|.bash_profile$|.bash_login$|.profile$|.bash_profile$|.bashrc$|.bash_logout$</field>
    <description>Unix shell config "$(file)" has been added</description>
    <mitre>
      <id>T1546.004</id>
    </mitre>
  </rule>
```

Rule ini akan trigger ketika sebuah shell autorun config seperti `.bash` atau `.sh` di beberapa folder seperti `/etc/bash.bashrc` telah dibuat.

![](media/Custom8TriggerAgent.png)

![](media/Custom8TriggerManager.png)


### Rule 9
```XML
  <rule id="100009" level="10">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">\/etc\/profile$|\/etc/profile.d\/|\/etc\/bash.bashrc$|\/etc\/bash.bash_logout$|.bash_profile$|.bash_login$|.profile$|.bash_profile$|.bashrc$|.bash_logout$</field>
    <description>Unix shell config "$(file)" has been modified</description>
    <mitre>
      <id>T1546.004</id>
    </mitre>
  </rule>
```

Rule ini akan trigger ketika sebuah shell autorun config seperti `.bash` atau `.sh` di beberapa folder seperti `/etc/bash.bashrc` telah dimodifikasi.

![](media/Custom9TriggerAgent.png)

![](media/Custom9TriggerManager.png)
