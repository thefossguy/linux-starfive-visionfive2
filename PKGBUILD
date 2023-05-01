pkgbase=linux-starfive-visionfive2
pkgver=6.4.arch1
pkgrel=1
pkgdesc='Linux'
url="https://github.com/torvalds/linux/"
arch=(riscv64)
license=(GPL2)
makedepends=(
  bc libelf pahole cpio perl tar xz gettext
  xmlto python-sphinx python-sphinx_rtd_theme graphviz imagemagick texlive-latexextra
  git
)
options=('!strip')
_srcname=archlinux-linux
source=('config')
sha512sums=('52d87b89a8af47b3d42a94f93e6853349d21c8258b8b85ca62f9edc1be43bd3b2b65c1a607cbf5020d3c4e038704a1425bd2c74b84d352d8d76a16dd7a46c08d')

if [ "$(uname -m)" != "riscv64" ]; then
  makedepends+=(riscv64-linux-gnu-gcc)
  export CARCH=riscv64
  export CROSS_COMPILE=riscv64-linux-gnu-
  export X_STRIP=riscv64-linux-gnu-strip
else
  export X_STRIP=strip
fi

export ARCH=riscv
export CFLAGS="-march=rv64imafdc_zicsr_zba_zbb -mcpu=sifive-u74 -mtune=sifive-7-series -O2 -pipe"
export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
  #[[ -d "eswin_6600u" ]] || git clone --depth 1 https://github.com/eswincomputing/eswin_6600u
  [[ -d "$_srcname" ]] || git clone --depth 1 --branch "master" \
    "https://github.com/torvalds/linux/" "$_srcname"
  cd $_srcname

  # cleanup before pulling
  make clean
  make mrproper

  # make sure we have the most up-to-date sources
  git fetch && git pull

  echo "Setting version..."
  #scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  echo "Setting config..."
  [[ -f .config ]] && rm -v .config
  make defconfig

  # ensure some necessary options are enabled
  NECESSARY_CONFIG_OPTIONS=("CONFIG_SOC_STARFIVE=y", "CONFIG_CLK_STARFIVE_JH7110_SYS=y", "CONFIG_PINCTRL_STARFIVE_JH7110_SYS=y", "CONFIG_SERIAL_8250_DW=y", "MMC_DW_STARFIVE", "CONFIG_DWMAC_STARFIVE=m")
  for CFG_OPTION in ${NECESSARY_CONFIG_OPTIONS[@]}; do
    grep "${CFG_OPTION}" .config 2> /dev/null || (echo "ERROR: config option ${CFG_OPTION} not found" && exit 1)
  done

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  # make kernel
  cd $_srcname
  make all -j$(nproc)

  # make the USB WiFi dongle driver
  #cd ../eswin_6600u
  #make KERNELDIR=../$_srcname KBUILDDIR=../$_srcname product=6600u
  #mv wlan_ecr6600u_usb.ko ../$_srcname/drivers/net/wireless/eswin/wlan_ecr6600u_usb.ko
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod initramfs)
  optdepends=('wireless-regdb: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE KSMBD-MODULE)
  replaces=(virtualbox-guest-modules-arch wireguard-arch)

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  echo "Installing dtbs..."
  make INSTALL_DTBS_PATH="$pkgdir/usr/share/dtbs/$kernver" dtbs_install
  make INSTALL_DTBS_PATH="$pkgdir/boot/dtbs" dtbs_install

  # remove build links
  [ -d "$modulesdir"/build ] && rm -rf "$modulesdir"/build
  [ -d "$modulesdir"/source ] && rm -rf "$modulesdir"/source
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/riscv" -m644 arch/riscv/Makefile
  cp -t "$builddir" -a scripts

  # I am not enabling CONFIG_DEBUG_INFO_BTF_MODULES because it causes a build time error
  # that I have not had time to dive into _yet_. Disable it again.
  # required when DEBUG_INFO_BTF_MODULES is enabled
  #install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/riscv" -a arch/riscv/include
  install -Dt "$builddir/arch/riscv/kernel" -m644 arch/riscv/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */riscv/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
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

  echo "Stripping vmlinux..."
  $X_STRIP -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
