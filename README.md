gsdc.cvmfs
==========

CernVM-FS **클라이언트** 설치·설정 role.

[galaxyproject.cvmfs](https://github.com/galaxyproject/ansible-cvmfs)(0.5.x)의 클라이언트 동작을 옮기고
변수 이름을 그대로 유지했다. 그래서 기존 group_vars(`cvmfs_http_proxies`, `cvmfs_quota_limit` 등)를
고치지 않아도 같은 결과 파일(`/etc/cvmfs/default.local`, `domain.d/*.conf`)이 나온다.
여기에 사이트 playbook 마다 복사돼 있던 기능을 변수로 켜고 끄도록 넣었다.
기본값은 IPA·LVM 이 없는 일반 환경에서도 동작하도록 모두 꺼져 있다.

외부 role 을 그대로 쓰지 않는 이유: 버전을 고정하지 않으면 실행하는 곳마다 다른 버전이 설치돼
결과가 달라지고(컨트롤러 0.2.23 / 새 VM 0.5.2), stratum0/1·localproxy 등 쓰지 않는 기능이 많으며,
사이트별 후처리(캐시 LV, `.local` 설정, autofs, IPA automount)를 playbook 마다 따로 붙여야 했다.

요구 사항
---------
- EL 8/9/10 (dnf)
- `cvmfs_cache_lv` 를 쓸 때: `community.general`(lvol, filesystem), `ansible.posix`(mount) collection

변수
----

| 변수 | 기본값 | 설명 |
|---|---|---|
| `cvmfs_install_repo` | `true` | cernvm yum 저장소·GPG 키 관리 (OSG 저장소의 cvmfs 를 쓰면 `false`) |
| `cvmfs_packages` | `[cvmfs]` | 설치할 패키지 |
| `cvmfs_config_package` | `null` | 예: `cvmfs-config-default`, `cvmfs-config-osg` |
| `cvmfs_upgrade_client` | `false` | `true` 면 최신으로 업그레이드 |
| `cvmfs_http_proxies` | `[DIRECT]` | `CVMFS_HTTP_PROXY` |
| `cvmfs_quota_limit` | `4000` | `CVMFS_QUOTA_LIMIT` (MB) |
| `cvmfs_cache_base` | `/var/lib/cvmfs` | `CVMFS_CACHE_BASE` |
| `cvmfs_repositories` | `[]` | `[{repository, client_options: ["KEY=VAL"]}]` |
| `cvmfs_server_urls` | `[]` | `[{domain, urls, use_geoapi}]` → `domain.d/<domain>.conf` |
| `cvmfs_keys` | `[]` | `[{path, key, owner}]` |
| `cvmfs_params` | `{}` | `default.local` 끝에 `KEY="VALUE"` 추가 (예: `CVMFS_TIMEOUT: 10`) |
| `cvmfs_config_repo` | `{}` | config 저장소 설정 (galaxyproject.cvmfs 와 같은 형식) |
| `cvmfs_local_configs` | `{}` | 경로 → 내용. 예: `/etc/cvmfs/config.d/cms.cern.ch.local` |
| `cvmfs_absent_configs` | `[]` | 지울 경로 |
| `cvmfs_reload_on_change` | `true` | 설정이 바뀌면 `cvmfs_config reload` |
| `cvmfs_probe` | `false` | 마지막에 `cvmfs_config probe` |
| `cvmfs_cache_lv` | `false` | 캐시를 전용 LV 에 둔다 (VG 는 `cvmfs_cache_lv_vg` → `cache` → `var` LV 의 VG 순) |
| `cvmfs_cache_lv_mb` | `quota*1.2+1024` | LV 크기 (MB) |
| `cvmfs_autofs_restart_dropin` | `false` | autofs `Restart=on-failure` + 죽은 `/cvmfs`(fd=-1) lazy umount drop-in |
| `cvmfs_nsswitch_automount` | `""` | 예: `"sss files"`. 비우면 nsswitch.conf 를 건드리지 않음 |
| `cvmfs_restart_autofs` | `false` | 마지막에 autofs 재시작 (작업이 도는 호스트에서는 권장하지 않음) |
| `cvmfs_ipa_automount_location` | `""` | 예: `cvmfs`. IPA automount location 을 이 값으로 전환 |
| `cvmfs_ipa_automount_replace` | `[gsdc]` | 바꿔도 되는 기존 location |
| `cvmfs_ipa_automount_remove_start_override` | `true` | 전환 전에 Foreman 의 autofs `start-override.conf` 제거 |

IPA automount 전환
------------------
`cvmfs_ipa_automount_location` 을 설정하면 패키지 설치 뒤에 다음을 **모두** 확인하고서만 전환한다.
location 의 `/cvmfs` 키가 `program:/etc/auto.cvmfs` 를 가리키므로, 그 프로그램 맵 없이 전환하면 시스템이 깨진다.

1. IPA 클라이언트이고 `ipa-client-automount` 가 있다
2. 해당 location 의 `auto.master` 에 `/cvmfs → program:/etc/auto.cvmfs` 키가 있다 (호스트 keytab 으로 조회)
3. `/etc/auto.cvmfs` 가 실행 가능한 파일로 있다
4. 현재 location 이 미설정이거나 `cvmfs_ipa_automount_replace` 에 있다 (이미 같으면 그대로)

`ipa-client-automount` 는 sssd·autofs 를 재시작한다. Foreman kickstart 의 autofs `start-override.conf`
(`automount -m` 에 sss 인스턴스가 보여야 통과)는 program 맵만 주는 location 과 양립하지 않아
autofs 재시작 루프를 만들므로 전환 전에 `/root/autofs-start-override.conf.removed` 로 옮긴다.

예시
----

```yaml
- hosts: wn
  become: true
  roles:
    - role: gsdc.cvmfs
      cvmfs_http_proxies:
        - http://squid1.example.org:3128
        - http://squid2.example.org:3128
      cvmfs_quota_limit: 40000
      cvmfs_cache_lv: true
      cvmfs_params:
        CVMFS_TIMEOUT: 10
        CVMFS_TIMEOUT_DIRECT: 10
      cvmfs_local_configs:
        /etc/cvmfs/config.d/cms.cern.ch.local: |
          export CMS_LOCAL_SITE=/cvmfs/cms.cern.ch/SITECONF/T3_KR_KISTI
      cvmfs_autofs_restart_dropin: true
      cvmfs_ipa_automount_location: cvmfs
```

License
-------
MIT. 클라이언트 설치·설정 부분은 galaxyproject/ansible-cvmfs (MIT)에서 가져왔다.
