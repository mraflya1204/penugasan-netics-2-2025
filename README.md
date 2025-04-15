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

Perubahan permission file dapat menjadi indikator adanya upaya eskalasi privilege atau penyamaran file berbahaya. Dengan memonitor perubahan ini, sistem dapat lebih cepat mendeteksi aktivitas mencurigakan yang memodifikasi hak akses file penting.

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

Brute force adalah teknik umum untuk mencoba masuk ke sistem secara tidak sah. Rule ini bertujuan mendeteksi dan memberikan peringatan dini terhadap upaya login berulang yang gagal melalui SSH, yang bisa menjadi tanda serangan.

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

Perubahan permission pada shell script bisa menjadikan script yang sebelumnya tidak bisa dieksekusi menjadi executable, yang dapat digunakan untuk menjalankan perintah berbahaya secara otomatis. Ini penting untuk mendeteksi persistence attack.

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

Penambahan file `authorized_keys` secara diam-diam bisa menjadi metode attacker untuk mendapatkan akses persistensi ke mesin target tanpa melalui proses otentikasi normal.

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

Modifikasi pada `authorized_keys` dapat berarti penggantian atau penambahan kunci oleh pihak tidak sah. Ini dapat membuka pintu bagi akses backdoor tanpa sepengetahuan admin.

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

File konfigurasi SSH mengatur bagaimana koneksi dilakukan. Modifikasi bisa berarti membuka port baru, mengaktifkan login root, atau hal berbahaya lainnya yang berhubungan dengan remote access.

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

### Rule 7 (A Local Account has been Modified)
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
Rule ini akan trigger event level 10 ketika beberapa folder, mainly `/etc/shadow`, `/etc/gshadow`, `/etc/passwrd`, `/etc/group`, `/etc/login.defs` telah dimodifikasi. 

File seperti `/etc/passwd` dan `/etc/shadow` menyimpan informasi penting terkait user. Modifikasi bisa menjadi tanda pembuatan, perubahan, atau eskalasi akun oleh attacker.

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

### Rule 8 (A Shell Config is Added)
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

Shell config baru bisa digunakan untuk menanam perintah yang akan dijalankan saat user login. Ini adalah salah satu metode persistence yang sering digunakan attacker.

![](media/Custom8TriggerAgent.png)

![](media/Custom8TriggerManager.png)


### Rule 9 (A Shell Config is Modified)
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

Perubahan konfigurasi shell bisa berarti penambahan perintah yang dijalankan otomatis saat shell dibuka. Hal ini sering dilakukan oleh malware untuk persistensi.

![](media/Custom9TriggerAgent.png)

![](media/Custom9TriggerManager.png)

### Rule 10-11 Prequisites
Untuk rule 10 - 11, konfigurasi file `/var/ossec/etc/ossec.conf` dan tambahkan:
```XML
<ossec_config>
  <syscheck>
    <directories check_all="yes" realtime="yes">/etc/systemd/system/</directories>
    <directories check_all="yes" realtime="yes">/usr/lib/systemd/system/</directories>
    <directories check_all="yes" realtime="yes">/usr/local/lib/systemd/system/</directories>
    <directories check_all="yes" realtime="yes">/lib/systemd/system/</directories>
  </syscheck>
</ossec_config>
```

### Rule 10 (System Scheduling is Added)
```XML
  <rule id="100010" level="12">
    <if_sid>554</if_sid>
    <field name="file" type="pcre2">\/systemd\/system\/.*\.timer$|\/systemd\/system\/.*\.service$</field>
    <description>[Systemd "$(file)" has been added]: Possible task/job scheduling</description>
    <mitre>
      <id>T1053.006</id>
    </mitre>
  </rule>
```
Rule ini akan trigger ketika sebuah Scheduled Task ditambahkan melalui systemd timer. File-file yang dimodifikasi biasanya terdapat pada `/etc/systemd/system/`, `/usr/lib/systemd/system/`, `/usr/local/lib/systemd/system/`, dan `/lib/systemd/system/`.

Menambahkan service atau timer baru melalui systemd dapat menjadi cara attacker menjadwalkan script berbahaya agar dijalankan otomatis (persistence mechanism).

![](media/Custom10TriggerAgent.png)

![](media/Custom10TriggerManager.png)

### Rule 11 (System Scheduling is Modified)
```XML
  <rule id="100011" level="12">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">\/systemd\/system\/.*\.timer$|\/systemd\/system\/.*\.service$</field>
    <description>[Systemd "$(file)" has been modified]: Possible task/job scheduling</description>
    <mitre>
      <id>T1053.006</id>
    </mitre>
  </rule>
```

Rule ini akan trigger ketika sebuah Scheduled Task dimodifikasi melalui systemd timer. File-file yang dimodifikasi biasanya terdapat pada `/etc/systemd/system/`, `/usr/lib/systemd/system/`, `/usr/local/lib/systemd/system/`, dan `/lib/systemd/system/`.

Modifikasi terhadap systemd timer atau service bisa menjadi tanda bahwa attacker mengubah waktu eksekusi tugas berbahaya, atau mengganti executable-nya.

![](media/Custom11TriggerAgent.png)

![](media/Custom11TriggerManager.png)

### Rule 12 - 17 Prequisites
Pertama, install Docker pada Wazuh Agent

![](media/DockerInstall.png)

Kemudian aktifkan remote command agar Agent bisa mendapatkan perintah dari Server menggunakan `echo "logcollector.remote_commands=1" >> /var/ossec/etc/local_internal_options.conf`

Kemudian restart Wazuh Agent

Pada Wazuh Manager, buat Wazuh Agent Group bernama `container` dengan menggunakan perintah `/var/ossec/bin/agent_groups -a -g container -q`

Kemudian, assign Agent yang akan menjadi host dari Docker Container dengan menggunakan `/var/ossec/bin/agent_groups -a -i 001 -g container -q`

Pada `/var/ossec/etc/shared/container/agent.conf`, tambahkan line berikut:
```XML
<agent_config>
  <!-- Configuration to enable Docker listener module. -->
  <wodle name="docker-listener">
    <interval>10m</interval>
    <attempts>5</attempts>
    <run_on_start>yes</run_on_start>
    <disabled>no</disabled>
  </wodle>  

  <!-- Command to extract container resources information. -->
  <localfile>
    <log_format>command</log_format>
    <command>docker stats --format "{{.Container}} {{.Name}} {{.CPUPerc}} {{.MemUsage}} {{.MemPerc}} {{.NetIO}}" --no-stream</command>
    <alias>docker container stats</alias>
    <frequency>120</frequency>
    <out_format>$(timestamp) $(hostname) docker-container-resource: $(log)</out_format>
  </localfile>

  <!-- Command to extract container health information. -->
  <localfile>
    <log_format>command</log_format>
    <command>docker ps --format "{{.Image}} {{.Names}} {{.Status}}"</command>
    <alias>docker container ps</alias>
    <frequency>120</frequency>
    <out_format>$(timestamp) $(hostname) docker-container-health: $(log)</out_format>
  </localfile>
</agent_config>
```
Line-line tersebut akan menjalankan Docker Listener Module dan melakukan command pada Endpoint Agent sebagai information gathering.

Buat decoder file bernama `docker_decoders.xml` dalam `/var/ossec/etc/decoders/`, lalu isi dengan:
```xml
<!-- Decoder for container resources information. -->
<decoder name="docker-container-resource">
  <program_name>^docker-container-resource</program_name>
</decoder>

<decoder name="docker-container-resource-child">
  <parent>docker-container-resource</parent>
  <prematch>ossec: output: 'docker container stats':</prematch>
  <regex>(\S+) (\S+) (\S+) (\S+) / (\S+) (\S+) (\S+) / (\S+)</regex>
  <order>container_id, container_name, container_cpu_usage, container_memory_usage, container_memory_limit, container_memory_perc, container_network_rx, container_network_tx</order>
</decoder>

<!-- Decoder for container health information. -->
<decoder name="docker-container-health">
  <program_name>^docker-container-health</program_name>
</decoder>

<decoder name="docker-container-health-child">
  <parent>docker-container-health</parent>
  <prematch>ossec: output: 'docker container ps':</prematch>
  <regex offset="after_prematch" type="pcre2">(\S+) (\S+) (.*?) \((.*?)\)</regex>
  <order>container_image, container_name, container_uptime, container_health_status</order>
</decoder>
```

Pada Wazuh Agent, buat sebuah environment bagi container menggunakan `mkdir container_env && cd $_`

Kemudian, buat `docker_compose.yml` file yang akan melakukan actions otomatis guna testing.
```yml
version: '3.8'

services:
  db:
    image: postgres
    container_name: postgres-container
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 20s
      timeout: 5s
      retries: 1
    ports:
      - '8001:5432'
    dns:
      - 8.8.8.8
      - 9.9.9.9
    volumes:
      - db:/var/lib/postgresql/data
    networks:
      - network
    mem_limit: "512M"

  cache:
    image: redis
    container_name: redis-container
    restart: always
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 20s
      timeout: 5s
      retries: 1
    ports:
      - '8002:6379'
    dns:
      - 8.8.8.8
      - 9.9.9.9
    volumes:
      - cache:/data
    networks:
      - network
    mem_limit: "512M"

  nginx:
    image: nginx
    container_name: nginx-container
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "stat /etc/nginx/nginx.conf || exit 1"]
      interval: 20s
      timeout: 5s
      retries: 1
    ports:
      - '8003:80'
      - '4443:443'
    dns:
      - 8.8.8.8
      - 9.9.9.9
    networks:
      - network
    mem_limit: "512M"

volumes:
  db: {}
  cache: {}
networks:
  network:
```
Docker Compose di atas akan melakukan pull image Nginx, Redis, dan Postgres dari Docker Hub, yang kemudian akan masing masing dijalnkan. Kemudian, environment akan membuat dan konek ke jaringan `container_env_network`. Setelah itu volume `container_env_db` dan `container_env_cache` akan dimount.

![](media/DockerRunning.png)

Ketika akan melakukan testing, jalankan command `docker exec -it nginx-container /bin/bash` untuk masuk ke shell dari container yang sedang berjalan. Kemudian jalankan command `apt update && apt install stress-ng -y` untuk melakukan install stress test pada container

![](media/StressInstall.png)


### Rule 12 (Container Resource Information)
```XML
  <!-- Rule for container resources information. -->
  <rule id="100100" level="5">
    <decoded_as>docker-container-resource</decoded_as>
    <description>Docker: Container $(container_name) Resources</description>
    <group>container_resource,</group>
  </rule>
```

Untuk trigger event ini, cukup masuk kedalam salah satu container dan trigger sebuah command.

![](media/Custom12TriggerAgent.png)

![](media/Custom12TriggerManager.png)

### Rule 13 (Container Memory and CPU Usage Warning)
```XML
  <rule id="100101" level="12">
    <if_sid>100100</if_sid>
    <field name="container_cpu_usage" type="pcre2">^(0*[8-9]\d|0*[1-9]\d{2,})</field>
    <field name="container_memory_perc" type="pcre2">^(0*[8-9]\d|0*[1-9]\d{2,})</field>
    <description>Docker: Container $(container_name) CPU usage ($(container_cpu_usage)) and memory usage ($(container_memory_perc)) is over 80%</description>
    <group>container_resource,</group>
  </rule>
```

Untuk trigger event ini, jalankan command `stress-ng -c 1 -l 80 -vm 1 --vm-bytes 500m -t 3m` untuk melakukan stress terhadap cpu dan memory dari container.

![](media/Custom13TriggerAgent.png)

![](media/Custom13TriggerManager.png)

### Rule 14 (Container CPU Usage Warning)
```XML
  <rule id="100102" level="12">
    <if_sid>100100</if_sid>
    <field name="container_cpu_usage" type="pcre2">^(0*[8-9]\d|0*[1-9]\d{2,})</field>
    <description>Docker: Container $(container_name) CPU usage ($(container_cpu_usage)) is over 80%</description>
    <group>container_resource,</group>
  </rule>  
```

Untuk trigger event ini, jalankan command `stress-ng -c 1 -l 80 -t 3m` untuk melakukan stress terhadap cpu dari container.

![](media/Custom14TriggerAgent.png)

![](media/Custom14TriggerManager.png)

### Rule 15 (Container GPU Usage Warning)
```XML
  <!-- Rule to trigger when container memory usage is above 80%. -->
  <rule id="100103" level="12">
    <if_sid>100100</if_sid>
    <field name="container_memory_perc" type="pcre2">^(0*[8-9]\d|0*[1-9]\d{2,})</field>
    <description>Docker: Container $(container_name) memory usage ($(container_memory_perc)) is over 80%</description>
    <group>container_resource,</group>
  </rule>
```

Untuk trigger event ini, jalankan command `stress-ng -vm 1 --vm-bytes 500m -t 3m` untuk melakukan stress terhadap gpu dari container.

![](media/Custom15TriggerAgent.png)

![](media/Custom15TriggerManager.png)

## Rule 16 (Container Health Information)
```XML
  <rule id="100104" level="5">
    <decoded_as>docker-container-health</decoded_as>
    <description>Docker: Container $(container_name) is $(container_health_status)</description>
    <group>container_health,</group>
  </rule>
```

Event ini akan trigger secara otomatis setiap 20s untuk cek health dari tiap container.

![](media/Custom16TriggerManager.png)

## Rule 17 (Unhealthy Container)
```XML
  <rule id="100105" level="12">
    <if_sid>100104</if_sid>
    <field name="container_health_status">^unhealthy$</field>
    <description>Docker: Container $(container_name) is $(container_health_status)</description>
    <group>container_health,</group>
  </rule>
```
Event ini akan trigger ketika container termasuk `unhealthy` atau ketika file `/etc/nginx/nginx.conf` tidak ada.

Untuk trip secara artifisial, cukup hapus file `/etc/nginx/nginx.conf`.

![](media/Custom17TriggerAgent.png)

![](media/Custom17TriggerManager.png)