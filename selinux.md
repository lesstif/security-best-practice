# 보안 강화 리눅스(Security-Enhanced Linux)

<!-- toc -->

**한 줄 요약**

> [!TIP] 
> SELinux 는 여러분을 불편하게 하지만 여러분의 시스템을 장악해서 악의적인 용도로 사용하려는 크래커들에게는 큰 좌절을 맛보게 하니 꼭 사용하세요.


SELinux 는 RHEL/CentOS 사용자들이 가장 증오하는 기능일 겁니다.
설치후 제일 먼저 하는 일이 SELinux 를 끄는 것이고 검색 엔진에서도 SElinux 를 치면 끄기가 자동 완성되니까요.

심지어 "리눅스에서 서비스를 운영하려면 SELinux를 꺼야 한다" 는 잘못된 지식이 널리 퍼져있습니다.

이것은 우리나라만이 아니라 외국도 마찬가지이며 SELinux 의 장점과 중단했을 때의  문제점을 널리 알리기 위한 [stop disabling SELinux](http://stopdisablingselinux.com/) 사이트도 생겨났습니다.

이 글은 널리 퍼져있는 SELinux 에 대한 잘못된 정보를 바로 잡고 사용을 장려하기 위해서 작성했습니다.

그러면 먼저 SELinux 를 이해하기 위해 필수적인 지식인 접근 통제에 대해서 알아 봅시다.

## 접근 통제

운영체제에서 접근 통제(Access Control)은 디렉터리나 파일, 네트워크 소켓 같은 시스템 자원을 적절한 권한을 가진 사용자나 그룹이 접근하고 사용할 수 있게 통제하는 것을 의미합니다.

접근 통제에서는 시스템 자원을 객체(Object)라고 하며 자원에 접근하는 사용자나 프로세스는 주체(Subject)라고 합니다.

즉 */etc/passwd* 파일은 객체이고 이 파일에 접근해서 암호를 변경하는 *passwd* 라는 명령어는 주체입니다.

### 임의 접근 통제

**임의 접근 통제**(DAC;Discretionary Access Control)는 시스템 객체에 대한 접근을 사용자나 또는 그룹의 신분을 기준으로 제한하는 방법입니다.

사용자나 그룹이 객체의 소유자라면 다른 주체에 대해 이 객체에 대한 접근 권한을 설정할 수 있습니다.

여기서 임의적이라는 말은 소유자는 자신의 판단에 의해서 권한을 줄 수 있다는 의미이며 구현이 용이하고 사용이 간편하기 때문에 전통적으로 유닉스나 윈도우등 대부분의 운영체제의 기본 접근 통제 모델로 사용되고 있습니다.

임의적 접근 통제는 사용자가 임의로 접근 권한을 지정하므로 사용자의 권한을 탈취당하면 사용자가 소유하고 있는 모든 객체의 접근 권한을 가질 수 있게 되는 치명적인 문제가 있습니다.

특히 **root** 계정은 모든 권한을 갖고 있으므로 root 권한을 탈취하면 시스템을 완벽하게 장악할 수 있으며 권한 탈취시 많이 사용되는 것은 아래에서 설명할 유닉스의 구조적인 2가지 보안 취약점입니다.


#### setuid/setgid 문제

사용자들의 암호는 /etc/shadow 에 저장되어 있으며 루트만 읽고 쓸수 있습니다.

하지만 사용자들은 *passwd* 명령어를 실행하여 자신의 암호를 변경할 수 있고 이때 */etc/shadow* 파일이 수정됩니다.

상대방 호스트가 동작하는지 확인하기 위해 사용하는 *ping* 은 ICMP(Internet Control Message Protocol) 패킷을 사용하므로 루트 권한이 필요하지만 일반 사용자도 ping 명령어를 사용하여 상대 호스트의 이상 여부를 확인할 수 있습니다.

유닉스 계열의 운영 체제는 실행 파일의 속성에 setuid(**set** **u**ser **ID** upon execution)또는 setgid(**set** **g**roup **ID** upon execution) 비트라는 것을 설정할 수 있으며 이 비트가 설정되어 있을 경우 해당 프로그램을 실행하면 실행 시점에 설정된 사용자(setuid), 또는 그룹(setgid) 권한으로 동작합니다.

즉 passwd 나 ping 같이 실행시 루트 권한이 필요한 프로그램은 소유자를 root 로 하고 setuid를 설정하면 실행 시점에 root 사용자로 전환되므로 root 만 가능한 동작을 수행할 수 있습니다.
 
 ```
 ls -l /bin/ping/ /usr/bin/passwd
 ```

![setuid 비트](https://www.lesstif.com/download/attachments/18219472/image2014-10-22%2023%3A28%3A26.png?version=1&modificationDate=1413987928000&api=v2 "setuid 비트")

파일의 사용자 퍼미션 부분의 's' 표시는 setuid 비트이며 그룹 퍼미션 부분에 's' 표시가 있을 경우 setgid 비트로 *passwd* 와 *ping* 은 setuid 비트만 설정되어 있는 것을 알수 있습니다.


하지만 setuid 비트가 붙은 프로그램에 보안 취약점이 있을 경우 공격자는 손쉽게 루트 권한을 획득할 수 있는 문제가 있습니다.

이 때문에 setuid 비트는 필요하지만 유닉스 시스템의 주요 보안 취약점이었으며 시스템 관리자의 골칫 덩어리였습니다.

시스템에 있는 setuid 비트가 붙은 프로그램은 다음 명령어로 찾을 수 있습니다.

```sh
find /bin /usr/bin /sbin -perm -4000 -exec ls -ldb {} \;
```

#### 잘 알려진 포트 daemon 문제

잘 알려진 포트(well-known port) 는 특정한 쓰임새를 위해서 IANA에서 할당한 TCP 및 UDP 포트 번호의 일부로 1024 미만의 포트 번호를 갖게 됩니다.

예로 웹에 사용되는 http(80), 메일 전송에 사용되는 smtp(25), 파일 전송에 사용되는 ftp(20, 21) 등이 잘 알려진 포트입니다.

전통적으로 잘 알려진 포트는 루트만이 사용할 수 있으므로 데몬 서비스는 모두 루트의 권한으로 기동됩니다.

보안 문제는 여기에서 발생하며 루트로 구동되었으므로 만약 서비스 데몬이 보안 취약점이 있거나 잘못된 설정이 있을 경우 서비스 데몬을 통해서 공격자는 루트 권한을 획득하게 되며 시스템의 모든 자원에 접근이 가능해 집니다.

### 강제 접근 통제

강제 접근 통제(MAC; Mandatory Access Control)는 미리 정해진 정책과 보안 등급에 의거하여 주체에게 허용된 접근 권한과 객체에게 부여된 허용 등급을 비교하여 접근을 통제하는 모델입니다.

높은 보안을 요구하는 정보는 낮은 보안 수준의 주체가 접근할 수 없으며 소유자라고 할 지라도 정책에 어긋나면 객체에 접근할 수 없으므로 강력한 보안을 제공합니다.

MAC 정책에서는 루트로 구동한 http 서버라도 접근 가능한 파일과 포트가 제한됩니다.
즉 취약점을 이용하여 httpd 의 권한을 획득했어도 */var/www/html*, */etc/httpd* 등의 사전에 허용한 폴더에만 접근 가능하며 80, 443, 8080 등의 포트만 접근이 허용되므로 ssh로 다른 서버로 접근을 시도하는등 이차 피해가 최소화 됩니다.

단점으로는 구현이 복잡하고 어려우며 모든 주체와 객체에 대해서 보안 등급과 허용 등급을 부여하여야 하므로 설정이 복잡하고 시스템 관리자가 접근 통제 모델에 대해 잘 이해하고 있어야 합니다.

## SELinux 란

SELinux는 미국 국가 안보국(NSA; National Security Agency; 또는 농담으로 No Such Agency 라고도 합니다.)에서 개발한 플라스크(Flask)라는 MAC 기반의 보안 커널을 리눅스에 이식한 커널 레벨의 보안 모듈입니다.

NSA는 구현한 소스를 리눅스 커뮤니티에 기증해서 2.6 버전부터 커널에 공식 포함되었습니다. <sup id="fnref1">[1](#footnote1)</sup>

RHEL 기반의 배포판에는 4 버전부터 공식적으로 포함되었으며 이제는 다양한 제품들이 SELinux를 지원하고 있으므로 기본적인 SELinux 개념과 설정을 알고 있다면 사용하는게 크게 어렵지는 않습니다.

> [!NOTE] 
> Android도 보안 문제가 대두되자 최신 버전부터는 SELinux 를 포팅한 SE Android 가 기본 탑재되어 있습니다.

**SELinux 는 기존 접근 통제 규칙보다 먼저 동작**하므로 SELinux 의 보안 정책에 맞지 않을 경우 차단해 버리게 됩니다.

이런 사실을 모를 경우 분명 파일의 소유자가 맞는데 *Permission Denied* 가 나거나 파일이 존재하는데 *File not Found* 등의 에러가 나는 원인을 찾지 못하고 SELinux 를 끄게 되고 정상 동작하는 것을 확인하고 SELinux 를 쓰면 안 된다는 생각을 갖게 됩니다.

이는 자동차 주행중에 경고등에 불이 들어왔다고 전구를 빼는 것과 비슷한 행동입니다.
실제 자동차에 문제가 생겼을 수도 있고 또는 경고등 자체가 고장났을수 있지만 가장 좋은 해결책은 원인을 찾고 적절한 조치를 취하는 것입니다. 

그동안 SELinux 에 대한 정보와 자료 부족때문에 그런 오해가 있었다면 이 글을 읽고 SELinux 에 대한 인식을 재고하고 왜 서비스가 SELinux 에 차단당했는지 원인을 찾고 적절한 조치를 취할 수 있는 계기가 되었으면 합니다.

### 장점

SELinux 를 사용할 경우 다음과 같은 장점이 있습니다.


1. **사전 정의된 접근 통제 정책 탑재**

 SELinux를 사용하면 사용자, 역할, 타입, 레벨등의 다양한 정보를 조합하여 어떤 프로세스가 어떤 파일, 디렉터리, 포트등에 접근 가능한지에 대해 사전에 잘 정의된 접근 통제 정책이 제공됩니다.

 그래서 MAC 적용을 위해 시스템 관리자가 할 일이 대폭 줄었고 애플리케이션의 변경없이 setuid와 1024 이하 포트를 사용하는 데몬을 안전하게 사용할 수 있습니다.

1. **"Deny All, Permit Some" 정책으로 잘못된 설정 최소화**

  서두에 설명했듯 "모든 걸 차단하고 필요한 것만 허용"하는 정책은 단순하면서 강력한 정보 보호를 위한 최선의 정책입니다. SELinux 의 보안 정책도 이 방식으로 사전에 설정되어 있으므로 잘못된 설정이 기본 포함돼 있을 여지가 적습니다.

1. **권한 상승 공격에 의한 취약점  감소**

  setuid 비트가 켜져 있거나 루트로 실행되는 프로세스처럼 위험한 프로그램들은 샌드박스안에서 별도의 도메인으로 격리되어 실행되므로 루트 권한을 탈취해도 해당 도메인에만 영향을 미치고 전체 시스템에 미치는 영향이 최소화됩니다.

  예로 아파치 httpd 서버의 보안 취약점을 통해 권한을 획득했어도 아파치같은 서버 데몬은 낮은 등급의 권한을 부여 받으므로 공격자는 일반 사용자의 홈 디렉터리를 읽을 수 없고 /tmp 임시 디렉터리에 파일을 쓸 수가 없습니다.

1. **잘못된 설정과 버그로부터 시스템 보호**

 잘못된 설정이나 신뢰할 수 없는 입력을 악용한 공격에서 프로세스를 보호할 수 있습니다. 

  예로 버퍼의 입력 길이등을 제대로 체크하지 않아서 발생하는 버퍼 오버 플로 공격(buffer overflow attack)의 경우 SELinux 는 어플리케이션이 메모리에 있는 코드를 실행할 수 없게 통제하므로 데몬 프로그램에 버퍼 오버 플로 버그가 있어도 쉘을 얻을 수가 없습니다.


### SELinux 의 한계

SELinux 의 주요 목표는 잘못된 설정이나 프로그램의 보안 버그로 인해 시스템이 공격 당해도 시스템과 데이타를 보호하고 2차 피해를 막는 것입니다. 

SELinux 는 여러 가지 보안 요소중에 하나이며 SELinux 로 모든 보안 요건이 충족되지는 않습니다. 

SELinux 는 침입 차단 시스템(IPS; Intrusion Protection System), 침입 탐지 시스템(IDS; Intrusion Detection System)이나 바이러스 백신이 아니므로 여러 보안 요소와 혼용하여 사용해야 합니다.

## SELinux 사용하기

### 동작 모드

SELinux는 그림과 같이 커널의 기본 기능으로 동작하며 모든 시스템 콜이 보안 정책을 확인해 접근 허용 여부를 판단하며  빠르게 처리하기 위해 보안 정책은 AVC(Access Vector Cache)라는 이름으로 커널 내부에서 캐싱합니다.

![SELinux 아키텍처](https://cloud.githubusercontent.com/assets/404534/12506805/d187db34-c134-11e5-85e3-76a71fd3ea9a.png "SELinux 아키텍처")


SELinux의 사용 여부는 *sestatus* 명령어를 통해 *Current mode* 에 enforcing 이 있는지 여부로 확인할 수 있습니다.

```
# sestatus

ELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28

```

SELinux 는 enforce, permissive, disable 3 가지 모드가 있으며 기본 설정은 *enforce mode*이고 보안 정책에 위배되는 모든 액션이 차단됩니다.

*permissive mode* 는 경고 메시지를 내고 차단하지는 않습니다.

모드의 전환은 *setenforce * 명령어를 사용하면 되며 옵션으로 1을 주면 **enforce mode** 로 0 일 경우 **permissive mode** 로 전환할 수 있습니다.

```
setenforce 0
```

![setenforce 0 결과](https://cloud.githubusercontent.com/assets/404534/14386181/be161840-fdde-11e5-86f8-603fccf0a14a.png "setenforce 0 결과")

마지막으로 disabled 는 SELinux 를 아예 사용하지 않도록 하는 설정입니다.

> **Danger** SELinux 를 끄는 것은 절대 권장하지 않지만 /etc/selinux/config 의 SELinux 항목을 아래와 같이 설정하고 재부팅을 하면 됩니다.

```
SELINUX=disabled
```


> [!WARNING] 
> SELinux 를 껐다가 다시 활성화하려면 재부팅이 필요하며 모든 리소스에 대해 보안 레이블을 추가해야 하므로 부팅 시간이 오래 걸릴 수 있습니다.


### Security Context 

SELinux 는 모든 프로세스와 객체마다 보안 컨텍스트(Security Context)<sup id="fnref2">[2](#footnote2)</sup>이라고 부르는 정보를 부여하여 관리하고 있습니다. 

이 정보는 접근 권한을 확인하는데 사용되고 있으며 SELinux 를 이해하기 위한 핵심 요소이며 다음 4가지 구성 요소로 이루어져 있습니다.

| 요소 | 설명 |
| -- | --  |
| 사용자 | 시스템의 사용자와는 별도의 SELinux 사용자이며 역할이나 레벨과 연계하여 접근 권한을 관리하는데 사용.  |
| 역할(Role) | 하나 혹은 그 이상의 타입과 연결되어 SELinux 의 사용자의 접근을 허용할 지 결정하는데 사용 |
| 타입(Type) | Type Enforcement의 속성중 하나로 프로세스의 도메인이나 파일의 타입을 지정하고 이를 기반으로 접근 통제를 수행 |
| 레벨(Level) | 레벨은 MLS(Multi Level System)에 필요하며 강제 접근 통제보다 더 강력한 보안이 필요할 때 사용하는 기능. 필자도 모릅니다.|

여기서 가장 중요하고 꼭 알아야 할 부분은 타입이며 이는 보안 컨텍스트의 핵심 부분으로 SELinux 를 활성화하면 파일이나 디렉터리등의 객체마다 보안 컨텍스트를 부여하고 있습니다.

SELinux 가 도입되면서 ls, ps, cp, mv 등의 유틸리티에는 *-Z, --context* 옵션이 추가되었습니다.

어떤 용도인지 알아보기 위해 *ls -ldz* 명령어를 실행해 봅시다.

```
ls -ldZ /var/www
```

![ls -ldZ 결과](https://cloud.githubusercontent.com/assets/404534/14387708/9a47acba-fde5-11e5-8c54-192db13c0fb0.png "ls -ldZ 결과")

*-Z* 를 붙이면 위에서 설명한 컨텍스트를 확인할 수 있으며 특히 타입(httpd_sys_content_t) 항목을 눈여겨 봅시다.

이제 *ps -Z|grep httpd*(또는 httpd 대신 nginx) 명령어를 실행해서 프로세스의 컨텍스트를 확인합시다.

```
ps -Z|grep httpd
```

![ps -Z 결과](https://cloud.githubusercontent.com/assets/404534/14387980/ccfcd81e-fde6-11e5-81f7-4660d3bdc8b4.png "ps -Z 결과")

httpd 프로세스는 httpd_t 라는 컨텍스트를 갖고 실행되는 것을 알 수 있습니다.

### Type Enforcement

TE(Type Enforcement)는 SELinux 의 기본적인 접근 통제를 처리하는 매커니즘으로 주체가 객체에 접근하려고 할 때 주체에 부여된 보안 컨텍스트가 객체에 접근할 권한이 있는지 판단하는 역할을 수행합니다. 

위의 예에서 아파치 웹 서버(httpd)가 /var/www/html/ 에 접근하려고 할 때 아파치 웹 서버는 주체가 되며 /var/www/html/ 는 객체가 되며 아파치 웹 서버에 부여된 보안 컨텍스트는 *httpd\_t* 가 됩니다.

객체에 부여된 보안 컨텍스트는 위에서 *ls -Z* 로 보았듯이 *httpd\_sys\_content\_t* 이며 httpd_t 는 httpd_sys_content_t 보안 컨텍스트가 부여된 객체에 접근이 허용되므로 아파치 웹 서버는 /var/www/html/ 디렉터리에 있는 컨텐츠를 읽을 수 있습니다.

SELinux 에 대한 오해는 바로 위의 Security Context 와 Type Enforcement 때문에 일어납니다.

즉 사전에 탑재된 httpd 에 대한 정책에 의하면 *httpd\_sys\_content\_t* 가 붙은 컨텐츠만 읽을 수 있고 /var/www 아래에 파일을 생성하면 자동으로 *httpd\_sys\_content\_t* 가 붙게 됩니다.

그러므로 /data/myweb-app 폴더를 만들고 이 안에 웹 서비스할 파일을 넣고 httpd 에 DocumentRoot 를 설정해도 SELinux 는 미리 허용된 경로가 아니므로 차단시켜서 *permission denied* 에러만 만나게 됩니다.

마찬가지로 httpd 가 접근할 수 있게 사전에 허용된 context 는 http_port_t 이며 *semanage* 명령어로 해당 포트 목록을 조회할 수 있습니다.

```sh
semanage port -l|grep http_port_t
```

기본적으로 허용된 포트는 아래와 같이 80, 443, 9000, 8000 등입니다.
```
http_port_t                    tcp      9004, 8000, 8080, 10080, 8001, 80, 81, 443, 488, 8008, 8009, 8443, 9000 
```

Type Enforcement로 인해 공격자가 "제로 데이 취약점"을 사용하여 httpd 의 권한을 획득해도 다른 서버로 ssh 연결을 할 수 없습니다. 22번 포트는 허용되지 않았기 때문이며 이로 인해 2차 피해를 최소화할 수 있습니다.

 또 다른 예로 파일 공유 서비스인 삼바와 mysql DBMS 가 같이 구동되는 서버에서 삼바를 공격하여 권한을 탈취했다고 가정해 봅시다.
 
 이제 공격자는 삼바를 통해 mysql 데이타베이스 파일을 가져가려고 시도할 수 있습니다.
 
 ```sh
 ls -lZ /var/lib/mysql/
 
-rw-rw----. mysql mysql system_u:object_r:mysqld_db_t:s0 ib_logfile0
-rw-rw----. mysql mysql system_u:object_r:mysqld_db_t:s0 ib_logfile1
-rw-rw----. mysql mysql system_u:object_r:mysqld_db_t:s0 ibdata1
 ```
 
  SELinux 하에서는 삼바와 mysql 은 별도의 도메인으로 격리되어 동작하며 mysql 데이타는 *mysqld_db_t* context 가 설정되어 있고 삼바는 해당 객체에 접근할 수 없습니다

하지만 위 정책으로 인해 웹 서버와 php-fpm 을 연동하는데 포트를 기본 fpm 포트(9000)가 아닌 9001을 사용했거나 톰캣과 연동하는데 8009, 8000이  아닌 포트를 사용했다면 SELinux 가 차단해서 서비스가 되지 않습니다.

### context 정보 얻기

SELinux의 문제를 해결하기 위해서는 context 정보를 확인하고 이를 맞춰주는게 매우 중요합니다.
이 작업을 용이하게 하기 위해 context 정보를 확인할 수 있는 유틸리티 사용법을 알아 봅시다.

먼저 다음 패키지를 설치합니다. 

```
yum install setools-console
```

#### seinfo

seinfo 는 policy를 조회하고 출력해주는 유틸리티입니다.

다음 명령어로 현재 정책을 요약해서 볼 수 있습니다.

```
seinfo
```

-a 옵션으로 조회할 속성을 지정할 수 있으며 다음 예제는 전체 도메인을 출력합니다. 

```
seinfo -adomain -x
```

#### sesearch

*sesearch* 는 정책에서 지정한 룰을 조회할 수 있는 유틸리티입니다.

다음 명령어는 httpd_sys_content_t 객체에 접근할 수 있는 롤 그룹을 표시합니다.

```
sesearch --role_allow -t httpd_sys_content_t 
```

--allow 옵션을 사용하면 특정 context에 허용된 액션을 알 수 있습니다.

```
sesearch --allow -s httpd_t
```

첫 번째 라인의 출력의 의미는 다음과 같습니다. 
 httpd_t 는 httpd_sys_content_t 가 설정된 파일에 대해 *ioctl, read, getattr, lock, open* system call 이 가능합니다. 
 즉 httpd_t 는 httpd_sys_content_t 컨텐츠를 읽을 수가 있습니다. 
 
```
 allow httpd_t httpd_sys_content_t : file { ioctl read getattr lock open } ; 
   allow httpd_t zoneminder_log_t : file { ioctl getattr lock append open } ; 
   allow httpd_t httpd_sys_content_t : dir { ioctl read getattr lock search open } ; 
   allow httpd_t zoneminder_log_t : dir { getattr search open } ; 
```

SELinux 는 위와 같이 잘 정의된 사전 정책을 탑재하고 있으므로 꼭 사용하는 것이 좋습니다.

### semanage

SELinux 에서 서비스가 안 도는 것은 보안 정책에 어긋나서이고 위에서 설명한 *seinfo, sesearch* 로 정책을 조회한 후에 조치해야 합니다.

예로 mysql을 3307 로 구동했다면 허용된 포트가 아니므로 web 서버가 mysql에 연결할 수 없으며 허용된 포트를 사용하거나 정책을 수정하는 [semanage](https://www.lesstif.com/pages/viewpage.action?pageId=18219476#SELinux사용하기-semange패키지) 명령어로 변경된 정보를 SELinux 에게 알려주면 됩니다.

*semanage*는 SELinux 의 보안 정책을 조회하고 추가/변경/삭제할 수 있는 명령행 기반의 유틸리티입니다.

 보안 컨텍스트는 파일이나 네트워크 포트, 네트워크 인터페이스등이며 이중에서 서비스 데몬이 SELinux 에서문제없이 동작하려면 꼭 알아 두어야 할 것이 파일과 네트워크 포트 컨텍스트입니다. 

semanage 는 아래 패키지를 설치해야 사용할 수 있습니다.

```
yum install -y policycoreutils-python
```

**포트 컨텍스트**

먼저 포트 컨텍스트 사용법을 익히기 위해 httpd 가 연결할 수 있는 포트 정보를  알아 보겠습니다. 

*semanage* 명령어의 첫 번째는 컨텍스트 이름이 와야 하며 이 경우 port 가 되며 오브젝트 리스트를 보는 -l 옵션을 추가하면 됩니다.

```
# semanage port -l|grep http_port_t

http_port_t                    tcp      8081, 8080, 8090, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

http_port_t 라는 컨텍스트는 tcp 프로토콜로 8080, 80 등의 포트에 연결이 허용된 것을 알수 있습니다.

만약 WAS 가 9876 포트를 사용할 경우 등록되지 않았으므로 SELinux 는 웹 서버가 9876 포트에 연결하지 못 하도록 차단하므로 정상적으로 서비스를 할 수 없습니다.

이 때 SELinux 를 끄지 말고 semanage 명령어로 포트를 추가해 주면 웹 서버가 9876 포트에 연결할 수 있습니다.

```
semanage port -a -p tcp -t http_port_t 9876
```

시스템에 따라 다음과 같은 에러가 발생하는 경우가 있습니다.

```
/usr/sbin/semanage: tcp/9876에 대한 포트가 이미 지정되었습니다
```

이는 9876 에 이미 할당된 context 가 있는 것이므로 포트를 추가하는 *-a* 대신 변경하는 *-m* 옵션을 사용해야 합니다.

```
semanage port -m -p tcp -t http_port_t 9876
```

포트를 삭제할 경우 *-d* 옵션을 사용하여 삭제할 프로토콜과 컨텍스트, 포트 번호를 명시해 주면 됩니다.

```
semanage port -d -p tcp -t http_port_t 9876
```

**파일 컨텍스트**

파일 컨텍스트는 fcontext 아규먼트를 추가하면 되며 웹 서버의 컨텐츠에 지정하는 httpd_sys_content_t가 할당된 파일 경로를 찾아 봅시다.

```
# semanage fcontext -l|grep httpd_sys_content_t

/var/lib/htdig(/.*)?                               all files          system_u:object_r:httpd_sys_content_t:s0 
/var/lib/trac(/.*)?                                all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www(/.*)?                                     all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www/icons(/.*)?                               all files          system_u:object_r:httpd_sys_content_t:s0 
/var/www/svn/conf(/.*)?                            all files          system_u:object_r:httpd_sys_content_t:s0 

```


아래에서 3번째 줄을 보면 */var/www(/.*)?* 로 지정되어 있는데 의미는 /var/www 하단에 생성되는 모든 파일과 폴더는 *httpd_sys_content_t* 를 붙이라는 의미입니다.

그러므로 웹 서버 컨텐츠가 /opt/mycontent 에 있다면 이후에 이 폴더에 생성한 모든 파일과 폴더에 *httpd_sys_content_t*  를 붙여야 정상적으로 서비스가 가능하므로 다음 semanage 로 fcontext 를 지정하면 됩니다.

```
semanage fcontext -a -t httpd_sys_content_t "/opt/mycontent(/.*)?"

```

## 문제 해결

SELinux 를 사용하려고 할 때 가장 큰 어려움중 하나는 왜 문제가 발생하는지 로그는 어디에 쌓이는지 잘 모르고 로그 메시지 해석이 어려운 점입니다.

SELinux 관련 로그는 */var/log/audit/audit.log* 에 저장되며 root 만 볼수 있습니다.

![SELinux 문제해결 절차](https://cloud.githubusercontent.com/assets/404534/15462125/69f01cee-20fb-11e6-993e-ff5e2047116e.png "SELinux 문제해결 절차")

이번 장에서는 로그를 검색하고 원인을 찾도록 도와주는 유틸리티에 대해서 알아 보겠습니다.


### audit2why

SELinux 의 audit 로그를 왜 차단되었는지 설명과 함께 출력하는 유틸리티입니다. 

사용은 다음과 같이 옵션으로 audit 파일의 경로를 입력해 주면 됩니다.

```
audit2why < /var/log/audit/audit.log 
```

결과는 아래와 같이 차단된 process 와 원인, 간단한 해결책을 같이 표시합니다.

```
type=AVC msg=audit(1463625135.602:32366): avc:  denied  { name_connect } for  pid=25862 comm="nginx" dest=9001 scontext=unconfined_u:system_r:httpd_t:s0 tcontext=system_u:object_r:tor_port_t:s0 tclass=tcp_socket

        Was caused by:
        The boolean httpd_can_network_connect was set incorrectly. 
        Description:
        Allow HTTPD scripts and modules to connect to the network using TCP.

        Allow access by executing:
        # setsebool -P httpd_can_network_connect 1
```

### ausearch

audit2why 는 모든 로그를 보여주므로 로그가 많이 쌓 여있을 경우 보기가 어렵습니다.
커널 로그를 검색할 수 있는 ausearch 명령어를 사용하면 기간별, 타입별로 로그를 조회할 수 있습니다.

> [!NOTE] 
> ausearch 를 사용하려면 아래 패키지를 설치해야 합니다.

```
yum install audit
```

*-m* 옵션으로 메시지 타입을 지정할 수 있으며 콤마를 구분자로 여러 개를 지정할 수도 있습니다.
*-ts, --start* 옵션으로 날자를 지정하면 해당 일 이후에 발생한 이벤트를 출력합니다.

다음 명령어는 2016년 5월 13일 이후에 발생한 이벤트만 표시합니다.

```
ausearch -m AVC,USER_AVC -ts 05/13/2016
```

> [!NOTE]
> 날자 지정 포맷은 locale 설정에 따라 달라지며 위와 같이 Month/Day/Year 형식으로 지정하려면 en_US 로 설정되어 있어야 합니다.

```
export LANG=en_US.utf8
```

또는 날자대신 now, recent, today, this-week, this-month, this-year 같은 문자열을 지정할 수 있습니다.

다음 명령어는 이번 달에 발생한 이벤트 로그를 출력합니다.

```
ausearch -m AVC,USER_AVC  -ts this-month
```

### sealert

audit2why 나 ausearch 가 출력하는 이벤트 로그는 익숙하지 않으면 해석과 조치가 어렵습니다. *sealert* 는 이를 쉽게 번역해 주는 유틸리티로 아래 패키지를 설치해야 사용할 수 있습니다.

```
yum install setroubleshoot-server
```

먼저 *-a* 옵션으로 현재 이벤트 로그를 분석할 수 있습니다.

```
sealert -a /var/log/audit/audit.log
```

분석이 끝나면 아래와 같이 차단 원인과 조치를 위한 명령어를 이해하기 쉽게 출력해 줍니다.

```
SELinux is preventing /usr/sbin/nginx from name_connect access on the tcp_socket .

*****  Plugin catchall_boolean (89.3 confidence) suggests  *******************

If you want to allow HTTPD scripts and modules to connect to the network using TCP.
Then you must tell SELinux about this by enabling the 'httpd_can_network_connect'boolean.
Do
setsebool -P httpd_can_network_connect 1
```

## 요약

- 웹 서버같이 [DMZ](firewall.html#비무장지대dmz)에 위치하는 서버는 꼭 SELinux 를 사용하는게 좋습니다.
- SELinux 보안 정책의 복잡도를 줄이기 위해 웹 서버는 static content 전송이나 Reverse Proxy 로만 사용하는 것을 권장하며 이럴 경우 SELinux 정책 수정이 거의 없이(필요시 port 컨텍스트 추가 정도) 사용할 수 있습니다.
- 서비스에 문제가 있고 SELinux 가 원인으로 짐작될 경우 끄는 것보다는 *setenforce 0* 로 잠시 허용 모드로 돌려 놓고 *audit2why, ausearch, sealert* 등을 활용하여 원인을 찾는 것이 좋습니다.


## 같이 읽기

* [SELinux 로 리눅스를 견고하게 하기](https://www.lesstif.com/pages/viewpage.action?pageId=18219470)
* [SELinux 에러 메시지 및 문제 해결](https://www.lesstif.com/pages/viewpage.action?pageId=12943496) - 자주 발생하는 에러와 해결책 모음
* [Basic SELinux Troubleshooting in CLI](https://access.redhat.com/articles/2191331)  - Redhat Customer Portal
* [cp/mv 와 SELinux security context](https://www.lesstif.com/pages/viewpage.action?pageId=14090283) - cp 와 mv 명령어가 SELinux 에서 어떻게 동작하는지 설명

<a name="footnote1" href="#fnref1">[1]</a> NSA 는 전 세계적인 감청망을 운용하고 있는 것으로 드러나서 SELinux 에 어떤 백도어가 있는 것 아니냐는 우려가 있을수 있지만 다행히 소스를 기증했기 때고 레드햇사와 리눅스 커뮤니티에서 오랫 기간 검증을 거쳐서 백도어 우려는 안 해도 됩니다.

<a name="footnote2" href="#fnref2">[2]</a> 보안 레이블(Security label) 이라고도 합니다.
