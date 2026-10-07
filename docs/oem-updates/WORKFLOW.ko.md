# OnePlus 13T / SM8750 디바이스 트리 반복 업데이트 가이드

이 가이드는 이 저장소의 실제 `.200` 보충 및 `.10.500` 통합 결과를 바탕으로 한다. [작업 기록](UPDATE-2026-10-07.ko.md), [현재 기준과 보류 항목](STATE.json), [파일 처리 목록](FILES.tsv), [검증 결과](VALIDATION.json)를 먼저 읽는다.

## 1. 다음번 시작 기준

| 항목 | 이번 작업 이후 기준 |
|---|---|
| 대상 저장소 | `android_kernel_oneplus_sm8750-devicetrees_los` |
| 대상 브랜치 | `lineage-24.0` (다음 작업에서도 실제 브랜치를 확인) |
| 현재 반영 릴리스 | `PKX110_16.0.10.500(CN01)` |
| 다음 OLD_OEM | `6a196f79f77acd55cf302e1696c038e4cad99254` |
| 이번 소스 커밋 | `287bb8e068eb9cec358a157e56ae58e80daa1584` |
| OEM 브랜치 | `oneplus/sm8750_b_16.0.0_oneplus_13t` |

OLD는 **검토·통합한 OEM 변경 범위의 기준**이다. generic Qualcomm, 다른 제품의 로컬 소스, 모듈 드라이버까지 모두 그 SHA라는 뜻이 아니다. 최신 HEAD를 무조건 NEW로 쓰지 않고 제품/지역/릴리스 제목을 확인한 전체 SHA로 고정한다.

새 대상 브랜치로 옮겼다면 이전 import가 실제 조상인지 먼저 확인한다. 이번 LOS는 `.200`이 없었기 때문에 보충 작업이 필요했다. 앞으로 `.500`이 반영된 같은 이력을 업데이트할 때는 옛 `.200` import를 다시 적용하지 않는다.

```bash
git -C "$CUSTOM" merge-base --is-ancestor \
  287bb8e068eb9cec358a157e56ae58e80daa1584 HEAD
```

실패하면 cherry-pick된 동등 변경인지, 아예 없는지 파일별로 확인한다. 커밋 SHA의 부재만으로 전체 재적용하지 않는다.

## 2. 작업 트리·SHA를 고정하고 자료 추출

아래는 이 workspace에서 쓰는 Bash 예시다. `NEW_OEM`은 확인한 40자리 SHA로 교체해야 실행된다. 실제 소스 수정은 하지 않고 `/tmp`의 새 디렉터리에 검토 자료를 만든다. 반복 실행하면 매번 별도 디렉터리가 생긴다.

```bash
set -euo pipefail
ROOT='/home/ccc/바탕화면/codex/pagani'
CUSTOM="$ROOT/android_kernel_oneplus_sm8750-devicetrees_los"
OEM="$ROOT/android_kernel_modules_and_devicetree_oneplus_sm8750"
OLD_OEM='6a196f79f77acd55cf302e1696c038e4cad99254'
NEW_OEM='여기에_확인한_40자리_SHA'

[[ "$OLD_OEM" =~ ^[0-9a-f]{40}$ ]]
[[ "$NEW_OEM" =~ ^[0-9a-f]{40}$ ]]
git -C "$OEM" cat-file -e "${OLD_OEM}^{commit}"
git -C "$OEM" cat-file -e "${NEW_OEM}^{commit}"
git -C "$OEM" show -s --format='%H%n%B' "$OLD_OEM" "$NEW_OEM"
git -C "$CUSTOM" branch --show-current
git -C "$CUSTOM" status --short

if [[ -n "$(git -C "$CUSTOM" status --porcelain)" ]]; then
    echo '기존 작업을 덮어쓰지 말고 별도 작업 공간을 준비하세요.' >&2
    exit 1
fi
CUSTOM_BASE=$(git -C "$CUSTOM" rev-parse HEAD)
WORK=$(mktemp -d /tmp/sm8750-dt-update.XXXXXX)
mkdir -p "$WORK/old" "$WORK/new"
SCOPE=(
    kernel_platform/qcom/opensource/devicetree
    vendor/qcom/opensource/audio-devicetree
    vendor/qcom/opensource/camera-devicetree
    vendor/qcom/opensource/display-devicetree
    vendor/qcom/opensource/wlan/wlan-devicetree
)
for rev in "$OLD_OEM" "$NEW_OEM"; do
    git -C "$OEM" ls-tree -r "$rev" -- "${SCOPE[@]}" > "$WORK/$rev.tree"
done
git -C "$OEM" archive "$OLD_OEM" "${SCOPE[@]}" | tar -xf - -C "$WORK/old"
git -C "$OEM" archive "$NEW_OEM" "${SCOPE[@]}" | tar -xf - -C "$WORK/new"
git -C "$OEM" diff --name-status --find-renames "$OLD_OEM" "$NEW_OEM" \
    > "$WORK/all-changes.txt"
git -C "$OEM" diff --name-status --find-renames "$OLD_OEM" "$NEW_OEM" \
    -- "${SCOPE[@]}" > "$WORK/dt-changes.txt"
git -C "$OEM" diff "$OLD_OEM" "$NEW_OEM" -- "${SCOPE[@]}" \
    > "$WORK/dt-changes.patch"
printf 'CUSTOM_BASE=%s\nOLD_OEM=%s\nNEW_OEM=%s\n' \
    "$CUSTOM_BASE" "$OLD_OEM" "$NEW_OEM" > "$WORK/revisions.txt"
printf '검토 자료: %s\n' "$WORK"
```

폴더가 이동·삭제되어 archive가 실패하면 그 경로 이동을 먼저 검토한다. `.gitattributes`의 export-ignore/export-subst가 있으면 archive와 Git 객체가 다를 수 있으므로 권위 있는 원본은 `git show "$NEW_OEM:경로"`다. 원본 checkout을 바꾸거나 사용자 변경을 자동 stash/reset하지 않는다.

브랜치가 다른 제품으로 바뀌면 `.200` 또는 `.500`을 그 제품의 공통 OEM 기준으로 가정하지 않는다. 해당 경로를 어느 제품/커밋에서 가져왔는지 따로 기록한다.

## 3. 범위 매핑과 의존성 추적

`P=kernel_platform/qcom/opensource/devicetree`, `V=vendor/qcom/opensource`.

| OEM 경로 | 대상 경로 | 처리 |
|---|---|---|
| P/oplus/ | oplus/ | 전체 비교, 실제 OEM delta 반영 및 로컬 예외 병합 |
| P/qcom/sun-debug.dtsi | qcom/sun-debug.dtsi | 로컬 차이와 OEM delta 함께 검토 |
| P/qcom/sun-pcie.dtsi | qcom/sun-pcie.dtsi | 동일 |
| P/qcom/sun-pmic-overlay.dtsi | qcom/sun-pmic-overlay.dtsi | 동일 |
| P/qcom/sun.dtsi | qcom/sun.dtsi | 변경 블록별 선택 병합 |
| V/audio-devicetree/oplus/ | qcom/audio/oplus/ | Oplus 변경 |
| V/camera-devicetree/oplus/ | qcom/camera/oplus/ | Oplus 변경 |
| V/display-devicetree/oplus/ | qcom/display/oplus/ | Oplus 변경 및 패널 로컬 예외 |
| V/display-devicetree/display/ | qcom/display/display/ | OEM 패널과 필요한 의존성만 |
| V/wlan/wlan-devicetree/oplus/ | qcom/wlan/oplus/ | Oplus 변경 및 XO trim 예외 |
| P/qcom/tuna-cdp-overlay-vw.dts | qcom/tuna-cdp-overlay-vw.dts | 이번에 추가한 VW include 의존성; 다음 delta에서도 추적 |
| P/qcom/tuna-pmiv0108-overlay-vw.dtsi | qcom/tuna-pmiv0108-overlay-vw.dtsi | 동일 |

`dsi-panel-A*`를 유일한 OEM 패널 판별 기준으로 쓰지 않는다. 전체 generic display 변경 목록에서 새 이름, 이동/삭제 및 include 참조를 확인한다. 이번 AA610 패널처럼 본체가 효과 파일을 include할 수 있다.

추가/수정 파일의 모든 quoted include를 재귀적으로 따라가고 실제 파일이 있는지 확인한다. `dt-bindings/...` angle include는 커널/모듈의 실제 헤더 경로로 해결한다. 이번 VW처럼 기본 범위 밖 파일이 필요하면 **그 의존 파일과 추가 이유**를 목록에 넣는다. 기존 generic 파일을 통째로 OEM 버전으로 교체해 해결하지 않는다.

새 속성·기능 ID는 OEM의 소비 코드도 확인한다. ROM에서 쓰는 실제 모듈의 구현·ID 값과 일치하는지는 별도 검사다. 동일 OEM 저장소에 구현이 있다는 사실만으로 로컬 ROM에 이미 있다는 결론을 내리지 않는다.

## 4. 세 방향 비교와 파일별 결정

각 파일에 `OLD_OEM`, `NEW_OEM`, `CUSTOM_BASE` 세 내용을 놓고 다음 기준으로 결정한다.

| 상황 | 처리 |
|---|---|
| OLD와 NEW 실질 내용 동일 | custom 유지. 이전 보류 항목은 별도 검토 |
| custom이 OLD와 같고 OEM만 변경 | 범위에 포함되는 일반 소스면 NEW 반영 가능 |
| custom과 OEM 모두 변경 | 로컬 변경 목적을 확인하고 블록별 병합 |
| OEM 신규 파일 | 동명 custom 파일·include·보드 참조 확인 후 추가 |
| OEM 삭제/이동 | 남은 custom 참조와 다른 보드의 필요성을 확인 후 처리 |
| CRLF/EOF/권한만 변경 | 유지하거나 별도의 정리 작업으로 분리 |

빌드 정의와 예외 파일은 단순 복사 대상에서 제외한다. 텍스트 3-way merge가 충돌 없이 끝나도 의미가 맞는지 확인한다. 파일 전체를 옛 버전으로 돌려 예외를 복구하면 새 OEM 내용을 잃을 수 있다.

검토표에 아래를 기록한다.

```text
대상 경로 / OEM 경로 / 기존 출처 / OLD→NEW 변경
반영·병합·유지·보류 / 결정 이유 / 관련 include·소비 코드
검증 결과 / 다음 작업으로 넘길 항목
```

이번과 같이 기준 import가 빠진 경우에는 먼저 해당 import 부모와 현재 custom을 대조한다. 다른 브랜치의 snapshot 전체를 복사하지 않는다. 참고 커밋과 원본 OEM 내용도 함께 비교하여 그 커밋의 실수나 누락까지 재현하지 않는다.

## 5. 반드시 유지·재검토할 예외

### WLAN

`qcom/wlan/oplus/{dodge,hummer,pagani}_sun-peach-cnss-v8-overlay.dtsi`의 `&wlan_peach`:

```dts
nvmem-cells = <0>;
nvmem-cell-names = "";
wcal-pbs-supply = <0>;
```

근거: `022598bf9d82ee9d9825e3372a1d6e04a44dc025`. 이미 있으면 중복 추가하지 않는다. 옛 파일의 wlan_direct_link 설정을 함께 되살리지 않는다. 부모/후속 overlay 변경 시 최종 조합에서 값이 덮이지 않는지 확인한다.

### ADSP/SLPI

`oplus/sun_overlay_common.dtsi`:

```dts
&adspslpi_mem {
    reg = <0x0 0x9ee00000 0x0 0x5080000>;
};
```

근거: `71e2754d403c67003597f8ca3e1ae100163f9bd0`. 공통 overlay 구조를 유지하고 기기별 옛 중복 정의로 되돌리지 않는다. 새 OEM이 메모리 맵을 바꾸면 DSP 펌웨어·예약 메모리·드라이버를 함께 검토해 값을 유지할지 갱신할지 결정한다.

### Qualcomm 및 다른 기기

- `sun.dtsi`: UFS PHY clock parent/symbol, `qcom,dmof`, bootargs의 `restrict_cma_redirect=false`, 공통 ADSP overlay 구조를 보호한다. enforce-high-irq-cpus 한 줄 삭제는 이번 결과이며 다음 작업의 고정 규칙이 아니다.
- AA590: `a52fefeebc1a64f8ac0a0678d647c0ad602af797`의 네 timing partial-update/ROI 비활성화를 보존한다.
- PASR: `450710a3aac3af54ef46504df2d46afecd6aa0eb`의 kera/parrot/pineapple/ravelin mem-offline 삭제를 되돌리지 않는다.
- Piloti/Pista 등 다른 제품에서 추가된 소스와 빌드 선택은 OEM에 없다는 이유로 삭제하지 않는다.

예외를 제거해야 한다면 근거와 검증 결과를 남기고 별도 변경으로 검토한다. 숫자나 코드를 영구 불변 규칙으로 취급하지 않는다.

## 6. 빌드 목록과 신규 기기 지원

점검 파일은 `oplus/Makefile`, `qcom/Makefile`, 각 하위 `Kbuild`, `qcom/camera/config/sun.mk`, `oplus/platform_map.bzl`이다. OEM으로 교체하지 않고 필요한 추가·삭제·이름·조건 변화만 반영한다.

- DTSI를 기존 DTS에서 include하는 경우 별도 DTBO 등록이 필요하지 않을 수 있다. 이번 Pagani MM 추가가 해당한다.
- DTBO 이름이 바뀌면 소비하는 make/Bazel/패키징 목록을 함께 추적한다.
- `CONFIG_PAGANI_DTB=y`에서 24821/24875만 선택되는지 실제 GNU make로 평가한다.
- Volkswagen 소스 존재, DTBO 등록, 이미지 포함, 실기 지원은 서로 다른 상태다. 현재 Volkswagen enablement는 보류 상태다.

기기 선택 목록 확인 예시:

```bash
cat > "$WORK/list.mk" <<'MAKE'
.PHONY: audit
audit:
	@printf '%s\n' '$(dtb-y) $(dtbo-y)'
MAKE
(
    cd "$CUSTOM"
    make --no-print-directory -s -f oplus/Makefile -f "$WORK/list.mk" \
        audit CONFIG_PAGANI_DTB=y
    make --no-print-directory -s -f qcom/camera/config/sun.mk \
        -f "$WORK/list.mk" audit CONFIG_ARCH_SUN=y
)
```

출력 이름에 대응하는 `.dts`가 있는지 확인한다. 다른 `CONFIG_*_DTB` 조합도 검사한다. 이 명령은 목록만 평가하며 DTB를 컴파일하거나 이미지에 패키징하지 않는다.

## 7. 형식과 모드 검사

변경 없는 파일의 CRLF/EOF를 건드리지 않는다. 실질 내용을 가져온 파일은 기존 LF 스타일에 맞추고 신규 일반 DTS/DTSI/빌드 파일만 0644로 추가한다. 기존 Git 100644/100755 모드는 보존한다. 전체 chmod, dos2unix, rsync --delete는 정기 절차에서 제외한다.

```bash
git -C "$CUSTOM" status --short
git -C "$CUSTOM" diff --stat "$CUSTOM_BASE" --
git -C "$CUSTOM" diff --summary "$CUSTOM_BASE" --
git -C "$CUSTOM" diff --ignore-space-at-eol "$CUSTOM_BASE" --
git -C "$CUSTOM" diff --check "$CUSTOM_BASE" --
git -C "$CUSTOM" ls-files --others --exclude-standard
```

`--ignore-space-at-eol`은 검토 보조다. 의미상 동일성 증명이 아니며 untracked 파일도 자동 포함하지 않는다. 승인한 경로를 명시해서 stage한 다음 반드시 staged 검사도 수행한다. `git add .`로 무관한 파일을 섞지 않는다.

```bash
# REVIEWED_PATHS=(검토한_실제_상대경로 ...)
git -C "$CUSTOM" add -- "${REVIEWED_PATHS[@]}"
git -C "$CUSTOM" diff --cached --stat
git -C "$CUSTOM" diff --cached --summary
git -C "$CUSTOM" diff --cached --check
git -C "$CUSTOM" diff --cached
git -C "$CUSTOM" diff --exit-code -- "${REVIEWED_PATHS[@]}"
```

마지막 명령은 stage 후 해당 경로에 추가로 남은 unstaged 변경이 없는지 확인한다. 원본과 맞추기 위해 trailing whitespace를 무조건 보존할 필요는 없지만, 정리는 가져온 줄에 한정하고 데이터 배열의 숫자나 문자열을 바꾸지 않는다.

## 8. 검증은 단계별로 기록

1. **소스/의존성:** 신규 include, binding, symbol, 기능 ID와 실제 소비 드라이버를 확인한다.
2. **실제 ROM 빌드:** ROM manifest, 커널·모듈 SHA, 제품 lunch/config, 툴체인, DTC 버전과 실행한 빌드 명령을 기록한다. 이 저장소만의 `make dtbs`는 동작하지 않는다. 사용하는 ROM의 실제 kernel/DTBO 빌드 경로로 빌드한다.
3. **최종 조합:** 패키징/부트로더가 선택하는 base와 overlay 순서를 확인하고, 그 순서대로 조합한다.
4. **패키징:** `dtbo.img` 및 사용하는 DTB 컨테이너의 항목 수·ID·revision·custom field·산출물 해시를 실제 제품 설정과 대조한다.
5. **실기:** 정확한 기기/펌웨어/ROM 조합에서 아래 회귀 검사를 수행한다.

독립 조각 검사는 보조 수단이다. 실제 헤더가 없는 상수를 임의로 정의하지 않는다. `dtc -f` 강제 출력을 정상 빌드로 인정하지 않는다. 이번 작업의 18개 fragment 통과와 충전 2개 차단은 [작업 기록](UPDATE-2026-10-07.ko.md)에 그대로 남겨 두었다.

실제 산출물이 준비된 후 조합/값 조회 예시:

```bash
# BASE_DTB, FINAL_DTB는 실제 산출물 경로.
# ORDERED_DTBO는 제품의 적용 순서를 확인해 채운 Bash 배열.
fdtoverlay -i "$BASE_DTB" -o "$FINAL_DTB" "${ORDERED_DTBO[@]}"

# symbols가 보존된 검증용 DT인 경우
adsp_node=$(fdtget -t s "$FINAL_DTB" /__symbols__ adspslpi_mem)
fdtget -t x "$FINAL_DTB" "$adsp_node" reg
# 현 정책의 기대값: 0 9ee00000 0 5080000

wlan_node=$(fdtget -t s "$FINAL_DTB" /__symbols__ wlan_peach)
fdtget -t x "$FINAL_DTB" "$wlan_node" nvmem-cells
fdtget -t s "$FINAL_DTB" "$wlan_node" nvmem-cell-names
fdtget -t x "$FINAL_DTB" "$wlan_node" wcal-pbs-supply
# 현 정책의 기대값: 0, 빈 문자열, 0
```

symbols가 제거된 산출물은 역컴파일해 실제 노드 경로를 확인한다. 이름을 추측하거나 임의 base로 조합하지 않는다. 24821/24875를 각각 검사하고 최종 FRR RAM 설정과 충전 속성도 조회한다. overlay 단독의 `reg`를 확인한 것을 최종 DT 확인으로 기록하지 않는다.

| 실기 항목 | 기록할 내용 |
|---|---|
| Wi-Fi | 켜기/끄기, 연결·재연결, 사용 대역, 절전 복귀 |
| ADSP/audio | DSP 로드 오류, 스피커·마이크·통화·재생 및 관련 로그 |
| 카메라 | 앱 진입, 렌즈 전환, 촬영·녹화, 오류 로그 |
| 디스플레이 | 밝기·주사율·AOD·회전·화면 꺼짐/복귀; 해당 패널 식별 |
| 충전 | 사용 충전기/케이블/프로토콜, SOC·온도·전류, 종료·재연결, 새 정책과 드라이버 호환 |
| 절전 | suspend/resume, 재부팅, 비정상 wakeup·커널 오류 |

실기 검증을 위해 별도 지시 없이 플래시하지 않는다. 실행 못한 검사는 “미실행/환경 제한”으로 남기고 소스 반영 완료와 제품 배포 검증 완료를 구분한다.

## 9. 커밋과 다음 기준 갱신

이번처럼 커밋이 요청되어 있으면 검사 후 새 커밋으로 남긴다. 기존 이력 재작성과 push는 별도 요청 없이 하지 않는다. 소스 업데이트와 문서는 분리하면 소스 커밋 SHA를 기록하기 쉽다.

```text
treewide: Update changes to PKX110_<새 버전>(CN01)

Source: OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8750
Source-branch: oneplus/sm8750_b_16.0.0_oneplus_13t
Previous-OEM-revision: <전체 OLD SHA>
OEM-revision: <전체 NEW SHA>
Custom-base: <작업 시작 SHA>

Scope: <실제 반영 범위와 범위 확장 근거>
Preserved: <WLAN/ADSP/CLO/기기별 수정/빌드 구조>
Deferred: <보류 목록 및 이전 기준>
Validation: <수행한 검증, 실패/차단/미실행 검증>
```

커밋 메시지는 파일로 써서 `git commit -F 파일`로 전달한다. 개행을 문자열 이스케이프로 만들어 셸에 전달하지 않는다. 커밋 후 `git show --check <SHA>`, `git status --short`로 최종 상태를 확인한다.

매번 다음 기록을 갱신한다.

- `UPDATE-날짜.ko.md`: 제품/지역/릴리스, OLD/NEW/custom SHA, 변경 목적, 의존성, 검증, 결과 소스 SHA.
- `STATE.json`: 통합한 범위의 다음 OLD, 혼합 출처·예외, 미반영 항목과 이유. 새 릴리스에 변경이 없더라도 기존 보류 목록은 유지한다.
- `FILES.tsv`: 반영/유지/보류 경로와 이유 또는 관련 기록, 필요하면 최종 해시.
- `VALIDATION.json` 또는 로그: 실제 명령·도구·입력·결과, 환경 제한을 분리.

기준을 NEW로 갱신해도 Volkswagen enablement, generic 선택 차이, 드라이버 호환 및 미실행 빌드/실기 검증은 해결된 것으로 표시하지 않는다. 다음 작업자는 이 목록과 OLD→NEW delta를 함께 검토해야 한다.
