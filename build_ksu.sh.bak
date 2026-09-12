#!/usr/bin/env bash

# ==============================================================================
#                              INITIALIZATION
# ==============================================================================

set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=Variables.conf
source "${PROJECT_ROOT}/Variables.conf"

# ==============================================================================
#                                FUNCTIONS
# ==============================================================================

usage() {
  local exit_code="${1:-1}"
  cat <<EOF
Usage: $0 [FIRMWARE] [ROOT_SOLUTION] [OPTIONS]

Note: Before building, configure your firmware branches and AVB metadata 
in the Variables.conf file.

FIRMWARE (choose one):
  --stable       Build for Stable firmware branch
  --beta         Build for Beta firmware branch

ROOT SOLUTION (choose one):
  --ksu          Integrate KernelSU
  --ksun         Integrate KernelSU-Next (Experimental, requires manual fixes)
  --sukisu       Integrate SukiSU-Ultra (Experimental, requires manual fixes)
  --none         No root integration

OPTIONS:
  --no-features  Build without extra features (Baseband-guard, extra patches, networking configs)
  --no-susfs     Build without SusFS integration
  --save-cache   Skip cleaning the compiler cache (out/ folder) for faster recompilation
  -h, --help     Show this help message and exit
EOF
  exit "$exit_code"
}

log() {
  printf '\nℹ️ ==> %s\n' "$1"
}

apply_patch_file() {
  local target="$1"
  local patch_file="$2"
  local optional="${3:-0}"

  printf '  -> %s\n' "$(basename "$patch_file")"
  if [[ "$optional" == "1" ]]; then
    patch --batch -d "$target" -p1 < "$patch_file" || true
  else
    patch --batch -d "$target" -p1 < "$patch_file"
  fi
}

# ==============================================================================
#                            ARGUMENT PARSING
# ==============================================================================

TYPE_FIRMWARE=""
KSU_TYPE_FLAG=""
USE_SUSFS=1
USE_FEATURES=1
SAVE_CACHE=0

for arg in "$@"; do
  case $arg in
    -h|--help) usage 0 ;;
    --stable) TYPE_FIRMWARE="STABLE" ; FOLDER_KERNEL="stable_source" ;;
    --beta) TYPE_FIRMWARE="BETA" ; FOLDER_KERNEL="beta_source" ;;
    --ksu) KSU_TYPE_FLAG="KernelSU" ;;
    --ksun) KSU_TYPE_FLAG="KernelSU-Next" ;;
    --sukisu) KSU_TYPE_FLAG="SukiSU-Ultra" ;;
    --none) KSU_TYPE_FLAG="None" ; USE_SUSFS=0 ;;
    --no-susfs) USE_SUSFS=0 ;;
    --no-features) USE_FEATURES=0 ;;
    --save-cache) SAVE_CACHE=1 ;;
    *) echo "Unknown flag: $arg"; usage 1 ;;
  esac
done

if [[ -z "${TYPE_FIRMWARE}" || -z "$KSU_TYPE_FLAG" ]]; then
  usage
fi

if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  KERNEL="${PROJECT_ROOT}/kernel_pixel_6.1"
  FOLDER_KERNEL="stable_source"
  GKI_BRANCH="$STABLE_BRANCH"
elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  KERNEL="${PROJECT_ROOT}/kernel_pixel_6.12"
  FOLDER_KERNEL="beta_source"
  GKI_BRANCH="$BETA_BRANCH"
fi

KSU_TYPE="$KSU_TYPE_FLAG"
AOSP="$KERNEL/common/ack"
DEVICE_DEFCONFIG="$KERNEL/private/devices/google/shusky/shusky_defconfig"
GKI_DEFCONFIG="$AOSP/arch/arm64/configs/gki_defconfig"
GKI_VERSION="$(echo "$GKI_BRANCH" | cut -d'-' -f1,2)"
PATCH_DIR="${PROJECT_ROOT}/patches/susfs/$GKI_VERSION"

# ==============================================================================
#                              CLEAN WORKSPACE
# ==============================================================================

log "Clean workspace"
for mnt in "${PROJECT_ROOT}/kernel_pixel_6.1/common/ack" "${PROJECT_ROOT}/kernel_pixel_6.12/common/ack"; do
  if mountpoint -q "$mnt"; then
    sudo umount "$mnt" || {
      log "Ошибка: Не удалось размонтировать $mnt."
      exit 1
    }
  fi
done
rm -rf "${PROJECT_ROOT}/susfs4ksu" "${PROJECT_ROOT}/KPatch-Next" "${PROJECT_ROOT}/output" "${PROJECT_ROOT}/AnyKernel3" 2>/dev/null || true

for kernel_folder in stable_source beta_source; do
  if [[ -d "${PROJECT_ROOT}/$kernel_folder" ]]; then
    cd "${PROJECT_ROOT}/$kernel_folder"
    rm -rf Baseband-guard KernelSU KernelSU-Next KPatch
    git reset --hard HEAD
    git clean -fdx
  fi
done

for pixel_repo in kernel_pixel_6.1 kernel_pixel_6.12; do
  if [[ -d "${PROJECT_ROOT}/$pixel_repo" ]]; then
    cd "${PROJECT_ROOT}/$pixel_repo"
    git reset --hard HEAD
    if [[ "$SAVE_CACHE" == "0" ]]; then
      git clean -fdx
    else
      git clean -fdx --exclude=out/
    fi
  fi
done

# ==============================================================================
#                             PREPARE SOURCES
# ==============================================================================

log "Updating Google kernel repository ($FOLDER_KERNEL)..."
if [[ ! -d "${PROJECT_ROOT}/$FOLDER_KERNEL/.git" ]]; then
  log "Cloning $FOLDER_KERNEL from branch $GKI_BRANCH"
  git clone --depth=1 -b "$GKI_BRANCH" https://android.googlesource.com/kernel/common "${PROJECT_ROOT}/$FOLDER_KERNEL"
else
  log "Fetching latest commits for $FOLDER_KERNEL ($GKI_BRANCH)..."
  git -C "${PROJECT_ROOT}/$FOLDER_KERNEL" fetch --depth=1 origin "$GKI_BRANCH"
  git -C "${PROJECT_ROOT}/$FOLDER_KERNEL" reset --hard FETCH_HEAD
  log "Updated $FOLDER_KERNEL to latest commit on $GKI_BRANCH."
fi

log "Updating GrapheneOS device repository..."
if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  if [[ ! -d "$KERNEL/.git" ]]; then
    log "Cloning kernel_pixel_6.1 from tag $PIXEL_TAG"
    git clone --depth=1 -b "$PIXEL_TAG" https://gitlab.com/grapheneos/kernel_pixel_6.1 "$KERNEL"
  else
    CURRENT_TAG=$(git -C "$KERNEL" describe --tags --exact-match 2>/dev/null || echo "")
    if [[ "$CURRENT_TAG" != "$PIXEL_TAG" ]]; then
      log "PIXEL_TAG changed. Fetching new tag $PIXEL_TAG..."
      git -C "$KERNEL" fetch --depth=1 origin tag "$PIXEL_TAG"
      git -C "$KERNEL" checkout FETCH_HEAD
    else
      log "kernel_pixel_6.1 is already at tag $PIXEL_TAG. No update needed."
    fi
  fi
else
  if [[ ! -d "$KERNEL/.git" ]]; then
    log "Cloning kernel_pixel_6.12 from branch $PIXEL_SPACECRAFT_BRANCH"
    git clone --depth=1 -b "$PIXEL_SPACECRAFT_BRANCH" https://gitlab.com/grapheneos/kernel_pixel_6.12 "$KERNEL"
  else
    log "Fetching latest commits for kernel_pixel_6.12 ($PIXEL_SPACECRAFT_BRANCH)..."
    git -C "$KERNEL" fetch --depth=1 origin "$PIXEL_SPACECRAFT_BRANCH"
    git -C "$KERNEL" reset --hard FETCH_HEAD
    log "Updated kernel_pixel_6.12 to latest commit."
  fi
fi

cd "${PROJECT_ROOT}"
if [[ "$USE_SUSFS" == "1" ]]; then
  log "Cloning susfs4ksu branch gki-${GKI_VERSION}"
  git clone https://gitlab.com/simonpunk/susfs4ksu --single-branch -b "gki-${GKI_VERSION}"
  cd "${PROJECT_ROOT}/susfs4ksu"
  if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
    git checkout "${SUSFS_KSU_COMMIT_6_1}"
  elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
    git checkout "${SUSFS_KSI_COMMIT_6_12}"
  fi
fi

log "Mount kernel source"
sudo mount --bind "${PROJECT_ROOT}/$FOLDER_KERNEL" "$AOSP"

log "Formation of variables"
rollback_index="${TYPE_FIRMWARE}_rollback_index"
salt="${TYPE_FIRMWARE}_salt"
os_version="${TYPE_FIRMWARE}_os_version"
fingerprint="${TYPE_FIRMWARE}_fingerprint"
security_patch="${TYPE_FIRMWARE}_security_patch"

# ==============================================================================
#                            KERNEL CONFIGURATION
# ==============================================================================

log "Configure kernel"
tee -a "${DEVICE_DEFCONFIG}" "${GKI_DEFCONFIG}" > /dev/null <<EOF
CONFIG_THREAD_INFO_IN_TASK=y
# CONFIG_WERROR is not set
$([[ "$KSU_TYPE" != "None" ]] && echo "CONFIG_KSU=y")
$([[ "$USE_SUSFS" == "1" ]] && echo "CONFIG_KSU_SUSFS=y")
EOF

if [[ "$KSU_TYPE" == "KernelSU-Next" ]]; then
cat >> "${DEVICE_DEFCONFIG}" <<EOF
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
EOF
fi

# Отключаем проверку сортировки defconfig
if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  # В 6.1 проверка вызывается через POST_DEFCONFIG_CMDS в build.config
  sed -i -E 's/check_defconfig( && )?//g' "$AOSP"/build.config.gki*
elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  # В 6.12 проверка задаётся атрибутом правила в BUILD.bazel
  sed -i '/name = "kernel_aarch64",/a\    check_defconfig = "disabled",' "$AOSP/BUILD.bazel"
fi

# Обход ABI-защиты (удаление защищенных экспортированных символов)
if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  rm -rf "$AOSP"/android/abi_gki_protected_exports_*
  perl -pi -e 's/^\s*"protected_exports_list"\s*:\s*"android\/abi_gki_protected_exports_aarch64",\s*$//;' "$AOSP/BUILD.bazel"
elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  perl -pi -e 's/^\s*protected_module_names_list\s*=\s*":gki_(?:aarch64|x86_64)_protected_module_names",\s*$//;' "$AOSP/BUILD.bazel"

  # Runtime ABI bypass для 6.12 (обход жесткой проверки CRC модулей)
  TARGET_VERSION_C="$AOSP/kernel/module/version.c"
  sed -i '/bad_version:/{:a;n;/return 0;/{s/return 0;/return 1;/;b};ba}' "$TARGET_VERSION_C"
  
  if ! grep -A 5 "bad_version:" "$TARGET_VERSION_C" | grep -q "return 1;"; then
    log "Error: Patch failed! 'return 1;' not found at bad_version: in $TARGET_VERSION_C"
    grep -A 10 "bad_version:" "$TARGET_VERSION_C" || true
    exit 1
  fi
fi

# Удаляет надпись dirty и maybe-dirty из наименования ядра
sed -i 's/echo -n -dirty/echo -n ""/g' "$KERNEL/build/kernel/kleaf/workspace_status_stamp.py"
sed -i "/stable_scmversion_cmd/s/-maybe-dirty//g" "$KERNEL/build/kernel/kleaf/impl/stamp.bzl" 2>/dev/null || true
sed -i 's/-dirty//' "$AOSP/scripts/setlocalversion" 2>/dev/null || true

if [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  #sed -i 's/ifdef CONFIG_ANDROID_BINDER_IPC_RUST/ifneq (,1)/' "$AOSP/drivers/android/binder/Makefile"
  #sed -i '/rust_binder\.ko/d' "$AOSP/modules.bzl"
  # Создаём символическую ссылку на бинарник rust. В prebuilts репозитория 6.12 от Google он имеет другую версию.
  if [[ -d "$KERNEL/prebuilts/rust/linux-x86/1.82.0.p1" ]]; then
    ln -sfn 1.82.0.p1 "$KERNEL/prebuilts/rust/linux-x86/1.82.0.p2"
  fi
fi

KERNEL_VER="$(sed -n '2,4p' "$AOSP/Makefile" | grep -oE '[0-9]+' | paste -sd '.')"
log "Версия ядра: ${KERNEL_VER}"

# ==============================================================================
#                           ROOT SOLUTION SETUP
# ==============================================================================

if [[ "$KSU_TYPE" != "None" ]]; then
  if [[ "$KSU_TYPE" == "KernelSU" ]]; then
    # ---------------------------------------------------------
    # Setup: KernelSU
    # ---------------------------------------------------------
    log "Install KernelSU"
    cd "$AOSP"
    curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
    cd "$AOSP/$KSU_TYPE"
    git checkout "$KSU_COMMIT"

    log "Adding Signature"
    APK_SIGN_C="${AOSP}/${KSU_TYPE}/kernel/manager/apk_sign.c"
    sed -i '/#ifdef EXPECTED_SIZE2/i \    if (check_v2_signature(path, 897, "b2e20f9dc4520d5f93a2e6ae19eecff475739dc0062d148644ee5111622d039d")) {\n        return true;\n    }' "${APK_SIGN_C}"
    if ! grep -q "b2e20f9dc4520d5f93a2e6ae19eecff475739dc0062d148644ee5111622d039d" "${APK_SIGN_C}"; then
      log "Error: Failed to patch custom signature in apk_sign.c!"
      exit 1
    fi

  elif [[ "$KSU_TYPE" == "KernelSU-Next" ]]; then
    # ---------------------------------------------------------
    # Setup: KernelSU-Next
    # ---------------------------------------------------------
    log "Install KernelSU-Next"
    cd "$AOSP"
    ksun_branch=$([[ "$USE_SUSFS" == "1" ]] && echo "dev-susfs" || echo "dev")
    curl -LSs "https://raw.githubusercontent.com/pershoot/KernelSU-Next/${ksun_branch}/kernel/setup.sh" | bash -s "${ksun_branch}"
    cd "$AOSP/KernelSU-Next"
    git checkout "$KSU_NEXT_COMMIT"

  elif [[ "$KSU_TYPE" == "SukiSU-Ultra" ]]; then
    # ---------------------------------------------------------
    # Setup: SukiSU-Ultra
    # ---------------------------------------------------------
    log "Install SukiSU-Ultra"
    cd "$AOSP"
    curl -LSs "https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh" | bash -s builtin

    log "Adding Signature"
    APK_SIGN_C="${AOSP}/KernelSU/kernel/manager/apk_sign.c"
    awk -v size="${SIG_SIZE_SUKISU:-897}" -v sha="${SIG_HASH_SUKISU:-b2e20f9dc4520d5f93a2e6ae19eecff475739dc0062d148644ee5111622d039d}" '
      /^} apk_sign_keys\[\] = \{$/  { print; skip=1; next }
      skip && /^\};$/               { print "    { " size ", \"" sha "\" }, // Custom"; skip=0 }
      skip                          { next }
                                    { print }
    ' "${APK_SIGN_C}" > "${APK_SIGN_C}.tmp" && mv "${APK_SIGN_C}.tmp" "${APK_SIGN_C}"
  fi
fi

if [[ "$USE_SUSFS" == "1" ]]; then
  log "Apply SUSFS patches"
  cp "${PROJECT_ROOT}/susfs4ksu/kernel_patches/fs/"* "$AOSP/fs/"
  cp "${PROJECT_ROOT}/susfs4ksu/kernel_patches/include/linux/"* "$AOSP/include/linux/"
  apply_patch_file "$AOSP" "${PROJECT_ROOT}/susfs4ksu/kernel_patches/50_add_susfs_in_gki-${GKI_VERSION}.patch" 1
  
  if [[ "$KSU_TYPE" == "KernelSU" ]]; then
    apply_patch_file "$AOSP/$KSU_TYPE" "${PROJECT_ROOT}/susfs4ksu/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch"
  fi
fi

# ==============================================================================
#                        ADDITIONAL PATCHES & SECURITY
# ==============================================================================

# Применение патчей из папок patches/susfs в зависимости от версии susfs.
if [[ "$USE_SUSFS" == "1" ]]; then
  log "Patching Kernel Source from $PATCH_DIR"
  if [[ -d "$PATCH_DIR" ]]; then
    for patch_file in "$PATCH_DIR"/*.patch; do
      if [[ -f "$patch_file" ]]; then
        apply_patch_file "$AOSP" "$patch_file"
      fi
    done
  fi
fi

if [[ "${USE_FEATURES}" == "1" ]]; then
  log "Install Baseband-guard"
  cd "$AOSP"
  wget -O- https://github.com/vc-teahouse/Baseband-guard/raw/main/setup.sh | bash
  tee -a "${DEVICE_DEFCONFIG}" > /dev/null <<< CONFIG_BBG=y
  sed -i '/^config LSM$/,/^help$/{ /^[[:space:]]*default/ { /baseband_guard/! s/selinux/selinux,baseband_guard/ } }' "$AOSP/security/Kconfig"

  if [[ -f "${PROJECT_ROOT}/patches/features.sh" ]]; then
    source "${PROJECT_ROOT}/patches/features.sh"
  fi
fi

# ==============================================================================
#                               BUILD PROCESS
# ==============================================================================

log "Correction of the .sh script used for build"
if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  # Удаляем этап подписи .ko модулей, так как собирается только raw image
  sed -i --follow-symlinks '/sign_file=$(mktemp)/,$d' ${KERNEL}/tools/build_dist.sh
  # Меняем цель Bazel с пакета дистрибутива на ядро
  sed -i --follow-symlinks 's/${DEVICE}\/dist/kernel/' ${KERNEL}/tools/build_dist.sh
  sed -i --follow-symlinks 's/bazel" run/bazel" build/' ${KERNEL}/tools/build_dist.sh
elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  # В android16 цель ядра называется :${DEVICE}/kernel (заменяем /dist на /kernel в ${DEVICE_TARGET}/dist)
  sed -i --follow-symlinks 's/\/dist/\/kernel/' ${KERNEL}/tools/build_dist.sh
  # Меняем run на build. Целевой объект "kernel" требует только build.
  sed -i --follow-symlinks 's/bazel" run/bazel" build/' ${KERNEL}/tools/build_dist.sh
fi

log "Build kernel"
cd "${KERNEL}"
if [[ "${SAVE_CACHE}" == "0" ]]; then
  tools/bazel clean --expunge
fi

if [[ "${TYPE_FIRMWARE}" == "STABLE" ]]; then
  KLEAF_REPO_MANIFEST=aosp_manifest.xml ./build_shusky.sh --config=fast --lto=none --keep_going
elif [[ "${TYPE_FIRMWARE}" == "BETA" ]]; then
  export BUILD_NUMBER=$(shuf -i 10000000-99999999 -n 1)
  ./build_shusky.sh --config=fast --config=stamp --extra_git_project=common/ack --lto=none --keep_going
fi

# ==============================================================================
#                             BOOT.IMG PREPARATION
# ==============================================================================

log "Prepare boot.img"
DIST="$(find "${KERNEL}/out/bazel/output_user_root" -type d -name kernel_kbuild_mixed_tree)"
TMPDIR="$(mktemp -d)"
mkdir -p "${TMPDIR}/gki"

curl -fsSL 'https://android.googlesource.com/platform/system/tools/mkbootimg/+/refs/heads/main/mkbootimg.py?format=TEXT' | base64 -d > "${TMPDIR}/mkbootimg.py"
curl -fsSL 'https://android.googlesource.com/platform/system/tools/mkbootimg/+/refs/heads/main/gki/generate_gki_certificate.py?format=TEXT' | base64 -d > "${TMPDIR}/gki/generate_gki_certificate.py"
curl -fsSL 'https://android.googlesource.com/platform/external/avb/+/refs/heads/main-kernel/avbtool.py?format=TEXT' | base64 -d > "${TMPDIR}/avbtool.py"
: > "${TMPDIR}/gki/__init__.py"

lz4 -l -12 -f "${DIST}/Image" "${TMPDIR}/kernel"
: > "${TMPDIR}/ramdisk"

python3 "${TMPDIR}/mkbootimg.py" \
  --header_version 4 \
  --pagesize 4096 \
  --kernel "${TMPDIR}/kernel" \
  --ramdisk "${TMPDIR}/ramdisk" \
  --cmdline '' \
  --os_patch_level "${!security_patch}" \
  -o "$DIST/boot.img"

mkdir -p "${PROJECT_ROOT}/output"

# ==============================================================================
#                          KERNELSU-NEXT PATCHING
# ==============================================================================

if [[ "$KSU_TYPE" == "KernelSU-Next" ]]; then
  log "Prepare patched boot.img"
  mkdir -p "${PROJECT_ROOT}/KPatch-Next"
  cd "${PROJECT_ROOT}/KPatch-Next"
  gh release download --repo KernelSU-Next/KPatch-Next -p 'kpimg-linux' -p 'kptools-linux' --clobber
  chmod +x kptools-linux
  ./kptools-linux -p -i "$DIST/Image" -k kpimg-linux -o "$DIST/Image_patched"
  mv -f "$DIST/Image_patched" "$DIST/Image"

  gh release download v30.2 --repo topjohnwu/Magisk -p 'Magisk*.apk' --clobber
  unzip -oj Magisk*.apk lib/x86_64/libmagiskboot.so
  mv -f libmagiskboot.so magiskboot
  chmod +x magiskboot

  cp "$DIST/boot.img" "${PROJECT_ROOT}/output/"
  cd "${PROJECT_ROOT}/output"
  "${PROJECT_ROOT}/KPatch-Next/magiskboot" unpack boot.img
  cp -f "$DIST/Image" ./kernel
  "${PROJECT_ROOT}/KPatch-Next/magiskboot" repack boot.img boot_patched.img
  mv -f boot_patched.img boot.img
else
  cp "$DIST/boot.img" "${PROJECT_ROOT}/output/"
  cd "${PROJECT_ROOT}/output"
fi

# ==============================================================================
#                               AVB SIGNING
# ==============================================================================

python3 "${TMPDIR}/avbtool.py" add_hash_footer \
  --image "${PROJECT_ROOT}/output/boot.img" \
  --partition_name boot \
  --partition_size 67108864 \
  --hash_algorithm sha256 \
  --algorithm NONE \
  --rollback_index "${!rollback_index}" \
  --rollback_index_location 0 \
  --flags 0 \
  --salt "${!salt}" \
  --prop "com.android.build.boot.os_version:${!os_version}" \
  --prop "com.android.build.boot.fingerprint:${!fingerprint}" \
  --prop "com.android.build.boot.security_patch:${!security_patch}"

# ==============================================================================
#                          PACKAGING AND CLEANUP
# ==============================================================================

rm -rf *kernel* ramdisk* header* dtb* unknown*
rm -rf "${TMPDIR}"

BOOT_KSU_PART=$([[ "$KSU_TYPE" != "None" ]] && echo "${KSU_TYPE}_" || echo "")
BOOT_NAME="${KERNEL_VER}_${BOOT_KSU_PART}${TYPE_FIRMWARE,,}_boot.img"
mv boot.img "$BOOT_NAME"
7z a "${BOOT_NAME}.7z" "$BOOT_NAME"

rm -f "$BOOT_NAME"
printf '\nDone: output/%s.7z\n' "${BOOT_NAME}"
