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
