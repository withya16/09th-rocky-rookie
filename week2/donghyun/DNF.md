**RED HAT ENTERPRISE LINUX 10**

> **Reference:** [Managing software with the DNF tool (RHEL 10)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/index)

**DNF software management tool을 사용하여 RPM repositories의 콘텐츠 관리하기**

**DNF tool**을 사용하여 **RPM repositories**를 통해 배포된 콘텐츠를 검색, 설치 및 활용합니다.

```bash
mr8356@mr8356:~$ tree /etc/dnf/
/etc/dnf/
├── aliases.d
├── dnf.conf
├── modules.d
├── modules.defaults.d
├── plugins
│   ├── copr.conf
│   ├── copr.d
│   ├── debuginfo-install.conf
│   └── kpatch.conf
├── protected.d
│   ├── grub2-efi-aa64.conf
│   ├── grub2-tools-minimal.conf
│   ├── selinux-policy-targeted.conf
│   ├── setup.conf
│   ├── sudo.conf
│   ├── systemd.conf
│   └── yum.conf
├── usr-drift-protected-paths.d
└── vars
    ├── contentdir
    ├── rltype
    ├── sigcontentdir
    └── stream
```

---

### Chapter 1. RHEL 10의 콘텐츠 배포

Red Hat Enterprise Linux (RHEL) 10에서 소프트웨어는 다양한 **Repositories**를 통해 배포됩니다. 이러한 **Repositories**를 통해 어플리케이션과 핵심 컴포넌트에 접근하여 시스템을 안정적이고 최신 상태로 유지할 수 있습니다.

**1.1. Repositories**

RHEL 10 **Repositories**는 안정적이고 지원되는 환경에 필요한 특정 소프트웨어, 운영체제 기능 및 다양한 시스템 컴포넌트를 제공합니다. RHEL은 **BaseOS**, **AppStream**, **CodeReady Linux Builder**, **Supplementary** 등 다양한 **Repositories**를 통해 콘텐츠를 배포합니다. 또한 **High Availability**와 같은 **RHEL Add-on repositories**에서 특정 콘텐츠를 사용할 수 있습니다.

RHEL이 콘텐츠를 배포하는 데 사용하는 주요 **Repositories**는 다음과 같습니다:

- **BaseOS**: 모든 설치의 토대가 되는 기본 운영체제 기능의 핵심 세트로 구성됩니다. 이 콘텐츠는 **RPM format**으로 제공되며 이전 릴리스와 유사한 지원 조건이 적용됩니다.
- **AppStream**: 다양한 **Workloads** 및 **Use cases**를 지원하기 위한 추가 **User-space applications**, **Runtime languages**, **Databases**를 포함합니다.

> **중요**: **BaseOS**와 **AppStream** 콘텐츠 세트는 모두 RHEL에 필수적이며 모든 **RHEL subscriptions**에서 사용할 수 있습니다.
> 
- **CodeReady Linux Builder**: 모든 **RHEL subscriptions**에서 사용할 수 있습니다. 개발자가 사용할 수 있는 추가 **Packages**를 제공합니다. Red Hat은 **CodeReady Linux Builder repository**에 포함된 패키지를 지원하지 않습니다.

**1.2. Application Streams**

Red Hat은 **User-space components**를 여러 버전의 **Application Streams**로 제공하며, 이는 핵심 운영체제 패키지보다 더 자주 업데이트됩니다. 이를 통해 플랫폼의 기본 안정성이나 특정 배포에 영향을 주지 않으면서 RHEL을 유연하게 커스터마이징할 수 있습니다.

**Application Streams**는 다음과 같은 형식으로 제공됩니다:

- **RPM format**
- **Software Collections**

RHEL 10은 초기 **Application Stream** 버전을 **RPM**으로 제공하며, **dnf install command**를 사용하여 설치할 수 있습니다.

> **중요**: 각 **Application Stream**은 고유한 **Life cycle**을 가지며, RHEL 10의 **Life cycle**과 같거나 더 짧을 수 있습니다. 설치하려는 **Application Stream**의 버전을 항상 결정하고, 먼저 **RHEL Application Stream life cycle**을 검토하십시오.
> 

---

### Chapter 2. DNF 설정

**DNF tool**의 **Global settings** 및 **Repository options**를 구성하여 RHEL 10 시스템의 소프트웨어 관리를 커스터마이징하십시오. 이러한 **Parameters**를 조정하여 **Package downloads**를 최적화하고 환경이 특정 운영 요구 사항을 충족하도록 할 수 있습니다. **DNF** 및 관련 유틸리티의 설정은 `/etc/dnf/dnf.conf` 파일의 **[main] section**에 저장됩니다.

```bash
mr8356@mr8356:~$ cat /etc/dnf/dnf.conf
[main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
```

**2.1. 현재 DNF 설정 확인**

활성화된 **DNF configuration**을 확인하여 소프트웨어 관리 설정을 검증하십시오. 이러한 **Parameters**를 검토하면 시스템이 효율적인 **Package management**를 위해 올바른 **Global** 및 **Repository settings**를 사용하는지 확인할 수 있습니다.

`/etc/dnf/dnf.conf` 파일의 **[main] section**에는 명시적으로 설정된 항목만 포함됩니다. 그러나 설정되지 않아 기본값을 사용하는 항목을 포함한 **[main] section**의 모든 설정을 표시할 수 있습니다.

**절차**

전역 **DNF configuration** 표시:

`# dnf config-manager --dump`

**2.2. DNF main options 설정**

**DNF**가 작동하는 방식을 제어하려면 `/etc/dnf/dnf.conf` 파일의 **[main] section**에서 **Key-value pairs**를 구성하십시오.

**절차**

1. `/etc/dnf/dnf.conf` 파일을 편집합니다.
2. 요구 사항에 따라 **[main] section**을 업데이트합니다.
3. 변경 사항을 저장합니다.

**2.3. DNF plugins 활성화 및 비활성화**

**DNF tool**의 **Plugins**를 관리하여 기능을 확장하십시오. **Plugins**를 활성화하거나 비활성화하여 운영 요구 사항에 맞게 특정 기능을 활성화하거나 제거할 수 있습니다.

**DNF tool**에서 **Plugins**는 기본적으로 로드됩니다. 그러나 **DNF**가 로드하는 **Plugins**에 영향을 줄 수 있습니다. 설치된 모든 **Plugin**은 `/etc/dnf/plugins/` 디렉토리에 고유한 설정 파일을 가질 수 있습니다. 설정 파일 이름은 `<plugin_name>.conf` 형식입니다. 기본적으로 **Plugins**는 일반적으로 활성화되어 있습니다.

> **경고**: 문제를 진단하기 위한 용도로만 모든 **Plugins**를 비활성화하십시오. **DNF**는 **product-id** 및 **subscription-manager**와 같은 특정 **Plugins**가 필요하며, 이를 비활성화하면 RHEL이 **CDN**에서 소프트웨어를 설치하거나 업데이트할 수 없게 됩니다.
> 

**절차**

- 전역적으로 **DNF plugins** 로딩을 제어하려면 `/etc/dnf/dnf.conf` 파일의 **[main] section**에 **plugins parameter**를 추가합니다.
    - `plugins=1` (기본값): 모든 **DNF plugins** 로딩 활성화.
    - `plugins=0`: 모든 **DNF plugins** 로딩 비활성화.
- 특정 **Plugin**을 비활성화하려면 `/etc/dnf/plugins/<plugin_name>.conf` 파일의 **[main] section**에 `enabled=False`를 추가합니다.
- 특정 명령에 대해 모든 **DNF plugins**를 비활성화하려면 명령에 `-noplugins` 옵션을 추가합니다.
- 특정 명령에 대해 특정 **DNF plugins**를 비활성화하려면 `-disableplugin=<plugin_name>` 옵션을 사용합니다.
- 특정 명령에 대해 특정 **DNF plugins**를 활성화하려면 `-enableplugin=<plugin_name>` 옵션을 사용합니다.

**2.4. DNF operation에서 패키지 제외**

특정 소프트웨어가 설치되거나 업데이트되는 것을 방지하려면 **excludepkgs option**을 사용하여 **DNF operations**에서 패키지를 제외하십시오. `/etc/dnf/dnf.conf`의 **[main]** 또는 **Repository section**에 정의할 수 있습니다.

> **참고**: `--disableexcludes` 옵션을 사용하여 설정된 제외를 일시적으로 비활성화할 수 있습니다.
> 

**절차**

`/etc/dnf/dnf.conf` 파일에 다음 라인을 추가하여 패키지를 제외합니다:

`excludepkgs=<package_name_1>,<package_name_2> ...`

---

### Chapter 3. RHEL 콘텐츠 검색

**DNF tool**을 사용하여 RHEL 10 **AppStream** 및 **BaseOS repositories**의 콘텐츠를 검색하고 조사하십시오.

**3.1. 소프트웨어 패키지 검색**

**절차**

- 패키지의 이름이나 요약(**Summary**)에서 용어를 검색하려면: `$ dnf search <term>`
- 이름, 요약 및 설명(**Description**)에서 용어를 검색하려면: `$ dnf search --all <term>`
- 패키지 이름을 검색하고 출력 결과에 버전까지 표시하려면: `$ dnf repoquery <package_name>`
- 특정 파일을 어떤 패키지가 제공하는지 검색하려면: `$ dnf provides <file_name>`

```bash
mr8356@mr8356:~$ dnf search nginx
Last metadata expiration check: 0:18:52 ago on Sun 15 Mar 2026 10:54:24 AM KST.
========================================= Name Exactly Matched: nginx =========================================
nginx.aarch64 : A high performance web server and reverse proxy server
======================================== Name & Summary Matched: nginx ========================================
nginx-all-modules.noarch : A meta package that installs all available Nginx modules
nginx-core.aarch64 : nginx minimal core
nginx-filesystem.noarch : The basic directory layout for the Nginx server
nginx-mod-http-image-filter.aarch64 : Nginx HTTP image filter module
nginx-mod-http-perl.aarch64 : Nginx HTTP perl module
nginx-mod-http-xslt-filter.aarch64 : Nginx XSLT module
nginx-mod-mail.aarch64 : Nginx mail modules
nginx-mod-stream.aarch64 : Nginx stream modules
pcp-pmda-nginx.aarch64 : Performance Co-Pilot (PCP) metrics for the Nginx Webserver

mr8356@mr8356:~$ dnf search --all "web server"
Last metadata expiration check: 0:19:25 ago on Sun 15 Mar 2026 10:54:24 AM KST.
================================== Summary & Description Matched: web server ==================================
nginx.aarch64 : A high performance web server and reverse proxy server
pcp-pmda-weblog.aarch64 : Performance Co-Pilot (PCP) metrics from web server logs
python3-tornado.aarch64 : Scalable, non-blocking web server and tools
========================================= Summary Matched: web server =========================================
libcurl.aarch64 : A library for getting files from web servers
======================================= Description Matched: web server =======================================
freeradius.aarch64 : High-performance and highly configurable free RADIUS server
git-instaweb.noarch : Repository browser in gitweb
httpd.aarch64 : Apache HTTP Server
mod_auth_openidc.aarch64 : OpenID Connect auth module for Apache HTTP Server
varnish.aarch64 : High-performance HTTP accelerator
```

**3.2. 소프트웨어 패키지 나열**

**절차**

아키텍처, 버전 번호, 설치된 **Repository**를 포함하여 사용 가능한 모든 패키지의 최신 버전을 나열합니다:

`$ dnf list --all`

*(출력 결과에서 **Repository** 이름 앞의 `@` 기호는 해당 패키지가 현재 설치되어 있음을 나타냅니다.)*

또는 버전 번호와 아키텍처를 포함하여 사용 가능한 모든 패키지를 표시하려면:

`$ dnf repoquery`

- `-installed`: 설치된 패키지만 나열.
- `-available`: 사용 가능한 모든 패키지 나열.
- `-upgrades`: 최신 버전이 있는 패키지 나열.

```bash
mr8356@mr8356:~$ dnf repoquery nginx
Last metadata expiration check: 0:20:25 ago on Sun 15 Mar 2026 10:54:24 AM KST.
nginx-2:1.26.3-1.el10.aarch64

mr8356@mr8356:~$ dnf repoquery nginx -a
Last metadata expiration check: 0:20:39 ago on Sun 15 Mar 2026 10:54:24 AM KST.
nginx-2:1.26.3-1.el10.aarch64
```

**3.3. 패키지 정보 표시**

**Version**, **Release**, **Architecture**, **Package size**, **Description** 등의 세부 **Metadata**를 확인합니다.

**절차**

하나 이상의 사용 가능한 패키지에 대한 정보를 표시합니다:

`$ dnf info <package_name>`

```bash
mr8356@mr8356:~$ dnf info nginx
Last metadata expiration check: 0:21:24 ago on Sun 15 Mar 2026 10:54:24 AM KST.
Available Packages
Name         : nginx
Epoch        : 2
Version      : 1.26.3
Release      : 1.el10
Architecture : aarch64
Size         : 33 k
Source       : nginx-1.26.3-1.el10.src.rpm
Repository   : appstream
Summary      : A high performance web server and reverse proxy server
URL          : https://nginx.org
License      : BSD-2-Clause
Description  : Nginx is a web server and a reverse proxy server for HTTP, SMTP, POP3 and
             : IMAP protocols, with a strong focus on high concurrency, performance and low
             : memory usage.
```

또는 **Repository** 내 해당 이름을 가진 모든 패키지의 정보를 보려면: `$ dnf repoquery --info <package_name>`

**3.4. 패키지 그룹 나열**

**절차**

- 설치된 그룹과 사용 가능한 그룹 모두 나열: `$ dnf group list`
- 특정 그룹에 포함된 필수(**Mandatory**), 선택(**Optional**), 기본(**Default**) 패키지 나열: `$ dnf group info "<group_name>"`

**3.5. Repository 나열**

**절차**

시스템에 활성화된 모든 **Repositories** 나열:

`$ dnf repolist`

- `-disabled`: 비활성화된 **Repositories**만 나열.
- `-all`: 활성/비활성 모두 나열.

```bash
mr8356@mr8356:~$ dnf repolist
repo id                                        repo name
appstream                                      Rocky Linux 10 - AppStream
baseos                                         Rocky Linux 10 - BaseOS
extras                                         Rocky Linux 10 - Extras
```

**3.6. DNF 입력 시 Glob expressions 지정**

많은 **DNF commands**는 패키지 이름이나 파일 경로 대신 **Glob expressions**를 허용합니다. **Wildcard characters**를 사용하여 관련 소프트웨어를 효율적으로 검색할 수 있습니다.

**절차**

- 전체 **Glob expression**을 단일 또는 이중 따옴표로 묶으십시오: `# dnf provides "*/<file_name>"`
- **Wildcard characters** 앞에 백슬래시(`\`)를 붙여 이스케이프 처리하십시오: `# dnf provides \*/<file_name>`

---

### Chapter 4. RHEL 콘텐츠 설치

**4.1. 패키지 설치**

**DNF**는 패키지 설치 중에 **Package dependencies**를 자동으로 해결하고 설치합니다.

**절차**

- **Repositories**에서 패키지 설치: `# dnf install <package_name_1> <package_name_2> ...`
- 특정 **Architecture** 지정 설치: `# dnf install <package_name>.<architecture>`
- 파일 경로를 통한 설치: `# dnf install <path_to_file>`
- 로컬 **RPM file** 설치: `# dnf install <path_to_RPM_file>`

**4.2. 패키지 그룹 설치**

**절차**

패키지 그룹 설치: `# dnf group install <group_name_or_ID>`

---

### Chapter 5. RHEL 콘텐츠 업데이트

**5.1. 업데이트 확인**

설치된 패키지에 대해 사용 가능한 업데이트가 있는지 확인합니다:

`# dnf check-update`

**5.2. 패키지 업데이트**

> **중요**: **Kernel** 업데이트 시, **dnf**는 `dnf upgrade`와 `dnf install` 중 무엇을 사용하든 항상 새로운 커널을 설치합니다. 이는 `installonlypkgs` 설정 옵션에 정의된 패키지(예: `kernel`, `kernel-core`, `kernel-modules`)에만 적용됩니다.
> 

**절차**

- 모든 패키지 및 **Dependencies** 업데이트: `# dnf upgrade`
- 단일 패키지 업데이트: `# dnf upgrade <package_name>`
- 특정 패키지 그룹만 업데이트: `# dnf group upgrade <group_name>`

**5.3. 보안 관련 패키지 업데이트**

**절차**

- **Security errata**가 있는 최신 패키지로 업그레이드: `# dnf upgrade --security`
- 마지막 **Security errata** 패키지로 업그레이드: `# dnf upgrade-minimal --security`

---

### Chapter 7. RHEL 콘텐츠 제거

**7.1. 설치된 패키지 제거**

단일 패키지 또는 여러 패키지를 제거할 수 있습니다. 제거하려는 패키지에 사용되지 않는 **Dependencies**가 있는 경우 **DNF**는 이들도 함께 삭제합니다.

**절차**

`# dnf remove <package_name_1> <package_name_2> ...`

**7.2. 패키지 그룹 제거**

**절차**

그룹 이름 또는 **Group ID**를 사용하여 제거:

`# dnf group remove <group_name> <group_ID>`

---

```bash
mr8356@mr8356:~$ dnf group list
Last metadata expiration check: 0:24:15 ago on Sun 15 Mar 2026 10:54:24 AM KST.
Available Environment Groups:
   Server with GUI
   Server
   Custom Operating System
Installed Environment Groups:
   Minimal Install
Available Groups:
   Legacy UNIX Compatibility
   Smart Card Support
   Console Internet Tools
   Container Management
   Development Tools
   .NET Development
   Graphical Administration Tools
   Headless Management
   Network Servers
   RPM Development Tools
   Scientific Support
   Security Tools
   System Tools
mr8356@mr8356:~$ dnf group install "Development Tools"
Error: This command has to be run with superuser privileges (under the root user on most systems).
mr8356@mr8356:~$ sudo dnf group install "Development Tools"
[sudo] password for mr8356:
Last metadata expiration check: 0:19:03 ago on Sun 15 Mar 2026 11:00:08 AM KST.
Dependencies resolved.
===============================================================================================================
 Package                                    Architecture  Version                        Repository       Size
===============================================================================================================
Upgrading:
 binutils                                   aarch64       2.41-58.el10_1.2               baseos          6.7 M
 glibc                                      aarch64       2.39-58.el10_1.7               baseos          1.7 M
 glibc-common                               aarch64       2.39-58.el10_1.7               baseos          304 k
 glibc-gconv-extra                          aarch64       2.39-58.el10_1.7               baseos          1.7 M
 glibc-langpack-en                          aarch64       2.39-58.el10_1.7               baseos          648 k
Installing group/module packages:
```

---

### Chapter 9. Managing custom software repositories

조직 전용 소프트웨어나 제3자(**Third-party**) 소프트웨어에 대한 접근 권한을 RHEL 10 시스템에 제공하려면, **DNF tool**을 사용하여 **Custom repositories**를 추가하고 구성하십시오. 이러한 커스텀 소스를 관리함으로써 환경에 필요한 특화된 패키지를 설치하고 업데이트할 수 있습니다.

**Repository** 설정은 `/etc/dnf/dnf.conf` 파일이나 `/etc/yum.repos.d/` 디렉토리 내의 `.repo` 파일에서 수행할 수 있습니다.

`/etc/dnf/dnf.conf` 파일은 **[main] section**을 포함하며, 대괄호(`[]`) 안에 고유한 **Repository ID**가 적힌 하나 이상의 **Repository sections**를 가질 수 있습니다(예: `[<repository-ID>]`). 이 섹션들을 사용하여 **Repository**별 옵션을 설정하고 개별 **DNF repositories**를 정의할 수 있습니다. **Repository ID**는 반드시 고유해야 합니다. `/etc/dnf/dnf.conf`의 개별 **Repository section**에 정의된 값은 해당 **Repository**에 대해 **[main] section**에 설정된 값을 오버라이드(**Override**)합니다.

다른 프로그램이 **DNF configuration** 파일을 수정할 때 발생할 수 있는 문제를 방지하기 위해, `/etc/dnf/dnf.conf` 파일 대신 별도의 `.repo` 파일에 **Custom repositories**를 추가하는 것을 권장합니다.

`dnf config-manager --add-repo` 명령을 사용하여 시스템에 **DNF repository**를 추가할 수 있습니다. 이 명령으로 추가된 **Repositories**는 기본적으로 활성화(**Enabled**)됩니다. 또한 `dnf config-manager` 명령을 사용하여 **Repository**를 비활성화(**Disable**)할 수도 있습니다.

> **경고**: Red Hat의 인증 기반 **Content Delivery Network(CDN)**가 아닌 확인되지 않았거나 신뢰할 수 없는 소스에서 소프트웨어 패키지를 얻고 설치하는 것은 잠재적인 보안 위험이며, 보안, 안정성, 호환성 및 유지 관리 문제를 야기할 수 있습니다.
> 

### 절차 (Procedure)

1. 시스템에 **Repository** 추가:
`# dnf config-manager --add-repo <repository_URL>`
2. 이전 명령으로 생성된 `/etc/yum.repos.d/<repository_URL>.repo` 파일의 설정을 검토하고 필요한 경우 업데이트:
`# cat /etc/yum.repos.d/<repository_URL>.repo`
3. (선택 사항) 시스템에 추가된 **DNF repository** 비활성화:
`# dnf config-manager --disable <repository_ID>`
    - **예시**: `# dnf config-manager --disable docker-ce-stable`
4. **Repository** 다시 활성화:
`# dnf config-manager --enable <repository_ID>`
    - **예시**: `# dnf config-manager --enable docker-ce-stable`

### 도커 레포지토리 설치

```bash
mr8356@mr8356:~$ sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo

mr8356@mr8356:~$ ls /etc/yum.repos.d/
docker-ce.repo  rocky-addons.repo  rocky-devel.repo  rocky-extras.repo  rocky.repo

mr8356@mr8356:~$ cat /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://download.docker.com/linux/centos/$releasever/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://download.docker.com/linux/centos/$releasever/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```

### 비활성화

```bash
mr8356@mr8356:~$ sudo dnf config-manager --disable docker-ce-stable

mr8356@mr8356:~$ cat /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=0 # 1->0 으로 바뀐것 확인 가능!
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```

### 다시 활성화

```bash
mr8356@mr8356:~$ dnf config-manager --enable docker-ce-stable
Error: This command has to be run with superuser privileges (under the root user on most systems).

mr8356@mr8356:~$ cat /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```