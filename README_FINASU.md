# FinaSU Kernel Builder

GKI_KernelSU_SUSFS 빌드 파이프라인에서 FinaSU 전용으로 추린 버전입니다.

## 변경점
- FinaSU 소스: 기본은 https://github.com/LyraVoid/FinaSU 에서 가져옴. 워크플로 입력 `finasu_source`
  - `latest`(기본): master 최신 커밋 / `pinned`: 검증된 고정 커밋(commit_mode 따름)
  - 내 수정본을 쓰고 싶으면 `optional/finasu-src-snapshot`을 `finasu-src`로 이름 바꾸면 그 `kernel/`이 upstream을 덮어씀(스냅샷은 업로드한 FinaSU-master)
- `main.yml` : Root Flavor는 FinaSU만 선택 가능. USB Arsenal/fanotify/pids/cfs_bw/abi_mode 입력 제거(항상 꺼짐/check)

## 기본 옵션 (스크린샷과 동일)
SUSFS, NoMount, Baseband Guard, Networking, DroidSpaces, NTSync, Ptrace Patch, Unicode Fix, BPF Stack = ON / Performance = OFF
브랜드 `Wild`, 페이지 4k, commit_mode `verified`.

## 사용법
1. 이 폴더 내용을 GitHub 저장소(main)에 푸시
2. Actions → Build Kernels → Run workflow (Kernel Version은 하나만 선택 권장, 예: 6.12.x-android16)
3. FinaSU는 SUSFS가 필수이므로 SUSFS를 끄지 말 것

## GKI_FinaSU-dev에서 가져온 SUSFS 수정 (FinaSU 전용)
- `.github/actions/finasu-susfs/` + `.github/scripts/finasu_susfs/` (신규), `.github/actions/susfs/action.yml`에서 FinaSU일 때만 호출
  1. verify: FinaSU 내장 SUSFS 연결부(extension/susfs.c, Kconfig, 진입점 등) 확인, 어긋나면 즉시 실패
  2. SELinux 새 모델 적응: FinaSU에 fake_state가 없으므로 SUSFS 패치의 SELinux 코드를 backup_sepolicy + 8인자 `_with_policy`로 변환 (5.10/5.15/6.1은 필수, 6.6/6.12는 선언 인자 수 보정)
  3. sus_path 폴백: 호출자 네임스페이스에서 경로를 못 찾으면(/storage/emulated 등 FUSE) LOOP 방식으로 전환
  4. 5.10 컴파일 수정: statfs extern 선언, security.h include
  5. susfs_def.h 누락 가드
  6. SUSFS Inline 훅 백엔드: FinaSU의 check_susfs_inline.sh 통과 시 `CONFIG_KSU_HOOK_SUSFS_INLINE=y`, 아니면 Tracepoint로 폴백
- 가져오지 않은 것: ZRAM, KPM, Re-Kernel, 브랜딩, SukiSU/BakaSU 전용 처리, `.rej` 후처리(기존 파이프라인이 이미 다른 방식으로 처리)

## uname 문자열 고정 (`uname_override`)
- 워크플로 입력 `uname_override`(기본값 `6.12.69-android16-6-g5fd39602326e-ab16372235-4k`): `uname -r`이 이 문자열과 똑같이 나오도록 `scripts/setlocalversion`의 마지막 줄을 고정 echo로 교체함. scm 해시(`-g…`), Kleaf 빌드 id(`-ab…`), 페이지 크기(`-4k`)까지 전부 이 문자열을 따름
- 해당 빌드의 `KERNEL.SUBLEVEL-androidNN`(예: `6.12.69-android16`)으로 시작할 때만 적용됨. 안 맞는 빌드는 경고만 남기고 기존 `brand_name` 방식으로 진행(여러 버전 동시 빌드해도 안 깨짐). 비우면 끔
- 빌드 후 `Verify uname release string` 단계가 Image의 linux_banner를 읽어 기대값과 비교하고 로그에 둘 다 출력함(불일치는 경고, 빌드는 계속)
- `brand_name`/`keep_kmi`보다 우선함
- 6.12.69 외 다른 서브레벨을 빌드할 땐 입력을 그 버전의 공식 문자열로 바꿔야 함

## ColorOS 17 공식 config (`config/stock_defconfig`)
- OnePlus 15 ColorOS 17 공식 boot 이미지(6.12.69-android16-6-g5fd39602326e-ab16372235-4k)의 임베디드 config(IKCONFIG)를 그대로 넣음
- `stock-config` 액션은 이 파일이 있으면 AOSP 프리빌트 대신 이걸 `/proc/config.gz`에 넣음 → 헤더가 6.12.18이 아니라 실제 6.12.69로 보이고 uname과 일치
- 빌드에 실제 쓰이는 config는 바꾸지 않음(보이는 값만 교체). 6.12 외 다른 커널 계열을 빌드할 땐 이 파일을 지우거나 이름을 바꿀 것
