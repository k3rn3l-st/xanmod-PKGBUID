# Maintainer: Joan Figueras <ffigue at gmail dot com>
# Contributor: Torge Matthies <openglfreak at googlemail dot com>
# Contributor: Jan Alexander Steffens (heftig) <jan.steffens@gmail.com>
# Contributor: Yoshi2889 <rick.2889 at gmail dot com>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

##
## The following variables can be customized at build time. Use env or export to change at your wish
##
##   Example: env _microarchitecture=25 use_numa=n use_tracers=n use_pds=n makepkg -sc
##
## Look inside 'choose-gcc-optimization.sh' to choose your microarchitecture
## Valid numbers between: 0 to 42
## Default is: 0 => generic
## Good option if your package is for one machine: 42 => native
## pondre 42 para usar las instrucciones de mi cpu
if [ -z ${_microarchitecture+x} ]; then
  _microarchitecture=42
fi

## Disable NUMA since most users do not have multiple processors. Breaks CUDA/NvEnc.
## Archlinux and Xanmod enable it by default.
## Set variable "use_numa" to: n to disable (possibly increase performance)
##                             y to enable  (stock default)
if [ -z ${use_numa+x} ]; then
  use_numa=y
fi

## For performance you can disable FUNCTION_TRACER/GRAPH_TRACER. Limits debugging and analyzing of the kernel.
## Stock Archlinux and Xanmod have this enabled. 
## Set variable "use_tracers" to: n to disable (possibly increase performance)
##                                y to enable  (stock default)
if [ -z ${use_tracers+x} ]; then
  use_tracers=y
fi

## Enable Cachy CPU scheduler by default https://github.com/xanmod/linux/blob/5.8/Documentation/scheduler/sched-Cachy.rst
## Set variable "use_cachy" to: n to disable (stock Xanmod)
##                              y to enable
if [ -z ${use_cachy+x} ]; then
  use_cachy=n
fi

## Enable CONFIG_USER_NS_UNPRIVILEGED flag https://aur.archlinux.org/cgit/aur.git/tree/0001-ZEN-Add-sysctl-and-CONFIG-to-disallow-unprivileged-C.patch?h=linux-ck
## Set variable "use_ns" to: n to disable (stock Xanmod)
##                           y to enable (stock Archlinux)
if [ -z ${use_ns+x} ]; then
  use_ns=n
fi

# Compile ONLY used modules to VASTLYreduce the number of modules built
# and the build time.
#
# To keep track of which modules are needed for your specific system/hardware,
# give module_db script a try: https://aur.archlinux.org/packages/modprobed-db
# This PKGBUILD read the database kept if it exists
#
# More at this wiki page ---> https://wiki.archlinux.org/index.php/Modprobed-db
if [ -z ${_localmodcfg} ]; then
  _localmodcfg=y
fi

# Tweak kernel options prior to a build via nconfig
_makenconfig=y

### IMPORTANT: Do no edit below this line unless you know what you're doing

pkgbase=linux-xanmod
pkgver=5.9.13
_major=5.9
_branch=5.x
xanmod=2
pkgrel=${xanmod}
pkgdesc='Linux Xanmod'
url="http://www.xanmod.org/"
arch=(x86_64)

license=(GPL2)
makedepends=(
  xmlto kmod inetutils bc libelf cpio clang llvm lld
)
options=('!strip')
_srcname="linux-${pkgver}-xanmod${xanmod}"

source=("https://cdn.kernel.org/pub/linux/kernel/v${_branch}/linux-${_major}.tar."{xz,sign}
        "https://github.com/xanmod/linux/releases/download/${pkgver}-xanmod${xanmod}/patch-${pkgver}-xanmod${xanmod}.xz"
        choose-gcc-optimization.sh
        '0001-ZEN-Add-sysctl-and-CONFIG-to-disallow-unprivileged-CLONE_NEWUSER.patch')
validpgpkeys=(
    'ABAF11C65A2970B130ABE3C479BE3E4300411886' # Linux Torvalds
    '647F28654894E3BD457199BE38DBBDC86092693E' # Greg Kroah-Hartman
)

# Archlinux patches
_commits=""
for _patch in $_commits; do
    source+=("${_patch}.patch::https://git.archlinux.org/linux.git/patch/?id=${_patch}")
done

sha256sums=('3239a4ee1250bf2048be988cc8cb46c487b2c8a0de5b1b032d38394d5c6b1a06'
            'SKIP'
            '85f09e233425bda814f7727a946474f40adce258073174a6417acc917b4f5bf9'
            '2c7369218e81dee86f8ac15bda741b9bb34fa9cefcb087760242277a8207d511'
            '6c66dba73251440352f93ff32b72f5dd49536d0f17ef9347867660fd3a626991')

# If use_cachy=y then download cachy patch
if [ "$use_cachy" = "y" ]; then
   echo "Cachy branch is not ready yet..." && exit 1
   source+=("https://github.com/xanmod/linux/releases/download/${pkgver}-xanmod${xanmod}-cachy/patch-${pkgver}-xanmod${xanmod}-cachy.xz")
   sha256sums+=('16b29e9e30979c8ff5f25ee5d46f9a9fc601ebda2b4aeabeeb85953628d38f10')
fi

export KBUILD_BUILD_HOST=${KBUILD_BUILD_HOST:-archlinux}
export KBUILD_BUILD_USER=${KBUILD_BUILD_USER:-makepkg}
export KBUILD_BUILD_TIMESTAMP=${KBUILD_BUILD_TIMESTAMP:-$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})}
export make CC=clang
export HOSTCC=clang
export NM=llvm-nm
export AR=llvm-ar
export HOSTLD=ld.lld
export LD=ld.lld
export OBJCOPY=llvm-objcopy
export STRIP=llvm-strip
prepare() {
  cd linux-${_major}

  # Apply Xanmod patch
  if [ "$use_cachy" = "y" ]; then
    patch -Np1 -i ../patch-${pkgver}-xanmod${xanmod}-cachy
  else
    patch -Np1 -i ../patch-${pkgver}-xanmod${xanmod}
  fi

  msg2 "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux-xanmod}" > localversion.20-pkgname

  # Archlinux patches
  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    msg2 "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  # CONFIG_STACK_VALIDATION gives better stack traces. Also is enabled in all official kernel packages by Archlinux team
  scripts/config --enable CONFIG_STACK_VALIDATION

  # Enable IKCONFIG following Arch's philosophy
  scripts/config --enable CONFIG_IKCONFIG \
                 --enable CONFIG_IKCONFIG_PROC
  #anbox support
  scripts/config --enable CONFIG_ANDROID
  scripts/config --enable CONFIG_ANDROID_BINDER_IPC
  scripts/config --enable CONFIG_ANDROID_BINDERFS
  scripts/config --set-str CONFIG_ANDROID_BINDER_DEVICES "binder,hwbinder,vndbinder"

  # User set. See at the top of this file
  if [ "$use_tracers" = "n" ]; then
    msg2 "Disabling FUNCTION_TRACER/GRAPH_TRACER..."
    scripts/config --disable CONFIG_FUNCTION_TRACER \
                   --disable CONFIG_STACK_TRACER
  fi

  if [ "$use_numa" = "n" ]; then
    msg2 "Disabling NUMA..."
    scripts/config --disable CONFIG_NUMA
  fi

  if [ "$use_cachy" = "y" ]; then
    msg2 "Enabling Cachy CPU scheduler by default..."
    scripts/config --enable CONFIG_CACHY_SCHED
  fi

  if [ "$use_ns" = "n" ]; then
    msg2 "Disabling CONFIG_USER_NS_UNPRIVILEGED"
    scripts/config --disable CONFIG_USER_NS_UNPRIVILEGED
  fi

  # Let's user choose microarchitecture optimization in GCC
  sh ${srcdir}/choose-gcc-optimization.sh $_microarchitecture

  # This is intended for the people that want to build this package with their own config
  # Put the file "myconfig" at the package folder to use this feature
  # If it's a full config, will be replaced
  # If not, you should use scripts/config commands, one by line
  if [ -f "${startdir}/myconfig" ]; then
    if ! grep -q 'scripts/config' "${startdir}/myconfig"; then
      # myconfig is a full config file. Replacing default .config
      msg2 "Using user CUSTOM config..."
      cp -f "${startdir}"/myconfig .config
    else
      # myconfig is a partial file. Applying every line
      msg2 "Applying configs..."
      cat "${startdir}"/myconfig | while read -r _linec ; do
        if echo "$_linec" | grep "scripts/config" ; then
          set -- $_linec
          "$@"
        else
          warning "Line format incorrect, ignoring..."
        fi
      done
    fi
    echo
  fi

  make olddefconfig

  ### Optionally load needed modules for the make localmodconfig
  # See https://aur.archlinux.org/packages/modprobed-db
  if [ "$_localmodcfg" = "y" ]; then
    if [ -f $HOME/.config/modprobed.db ]; then
      msg2 "Running Steven Rostedt's make localmodconfig now"
      make HOSTCC=clang CC=clang LSMOD=$HOME/.config/modprobed.db localmodconfig
    else
      msg2 "No modprobed.db data found"
      exit
    fi
  fi

  make -s kernelrelease > version
  msg2 "Prepared %s version %s" "$pkgbase" "$(<version)"

  [[ -z "$_makenconfig" ]] || make nconfig

  # save configuration for later reuse
  cat .config > "${startdir}/config.last"
}

build() {
  cd linux-${_major}
  make all
}

_package() {
  pkgdesc="The Linux kernel and modules with Xanmod patches"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=('linux' 'linux-xanmod-git')
  replaces=('linux-xanmod-git')
  conflicts=('linux-xanmod-git')

  cd linux-${_major}
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  msg2 "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  msg2 "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for Xanmod Linux kernel"
  provides=('linux-headers' 'linux-xanmod-git-headers')
  replaces=('linux-xanmod-git-headers')
  conflicts=('linux-xanmod-git-headers')

  cd linux-${_major}
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  msg2 "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  msg2 "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  msg2 "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  msg2 "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  msg2 "Removing documentation..."
  rm -r "$builddir/Documentation"

  msg2 "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  msg2 "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  msg2 "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  msg2 "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
