buildarch=8
pkgbase=linux-r2s
_desc="NanoPi R2S"
pkgver=5.10.49
pkgrel=1
pkgdesc='Linux'
_srctag=v${pkgver%.*}-${pkgver##*.}
arch=(aarch64)
license=(GPL2)
makedepends=(
  bc perl tar xz
  git dtc inetutils
)
options=('!strip')
_srcname=linux-$pkgver
source=(
  https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/${_srcname}.tar.{xz,sign}
  '0001-linux-5.10.25-r2s.patch'
  'config'
  'linux.preset'
  '60-linux.hook'
  '90-linux.hook'
)
validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
  'A2FF3A36AAA56654109064AB19802F8B0D70FC30'  # Jan Alexander Steffens (heftig)
)
sha256sums=(
  'b0d16de7e79c272b01996ad8ff8bdf3a6a011bc0c94049baccf69f05dde3025e'
  'SKIP'
  '7d91d000f2fb068c90d0d935223da24e2b0cfbd01c0ca6b34570930861435f4a'
  '9004ae64f90bfffcca93ceff416ede9748484cfad0999bbae45357dfd4a00a24'
  '6837b3e2152f142f3fff595c6cbd03423f6e7b8d525aac8ae3eb3b58392bd255'
  '452b8d4d71e1565ca91b1bebb280693549222ef51c47ba8964e411b2d461699c'
  '71df1b18a3885b151a3b9d926a91936da2acc90d5e27f1ad326745779cd3759d'
)

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
  cd $_srcname

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting config..."
  cp ../config .config
  make olddefconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh
}

build() {
  cd $_srcname
  # build!
  unset LDFLAGS
  make $MAKEFLAGS Image Image.gz modules
  # Generate device tree blobs with symbols to support applying device tree overlays in U-Boot
  make DTC_FLAGS="-@" dtbs
}

_package() {
  pkgdesc="The Linux Kernel and modules - $_desc"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(WIREGUARD-MODULE)
  replaces=(wireguard-arch linux-armv8 linux-aarch64)
  conflicts=(linux)
  backup=("etc/mkinitcpio.d/$pkgbase.preset")
  install=$pkgname.install

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  KARCH=arm64

  #echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  #install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  mkdir -p "$pkgdir"/{boot,usr/lib/modules}

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}

  make INSTALL_DTBS_PATH="$pkgdir/boot/dtbs" dtbs_install
  cp arch/$KARCH/boot/Image{,.gz} "$pkgdir/boot"

  # now we call depmod...
  depmod -b "$pkgdir/usr" -F System.map "$kernver"

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|$pkgbase|g
    s|%KERNVER%|$kernver|g
  "

  # install mkinitcpio preset file
  sed "$_subst" ../linux.preset |
    install -Dm644 /dev/stdin "$pkgdir/etc/mkinitcpio.d/$pkgbase.preset"

  # install pacman hooks
  sed "$_subst" ../60-linux.hook |
    install -Dm644 /dev/stdin "$pkgdir/usr/share/libalpm/hooks/60-$pkgbase.hook"
  sed "$_subst" ../90-linux.hook |
    install -Dm644 /dev/stdin "$pkgdir/usr/share/libalpm/hooks/90-$pkgbase.hook"
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the Linux kernel - $_desc"
  provides=("linux-headers=$pkgver")
  conflicts=('linux-headers')

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/$KARCH" -m644 arch/$KARCH/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  #install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  mkdir -p "${builddir}/arch/arm"
  cp -t "$builddir" -a include scripts
  cp -t "$builddir/arch/$KARCH" -a arch/$KARCH/include
  cp -t "$builddir/arch/arm" -a arch/arm/include
  install -Dt "$builddir/arch/$KARCH/kernel" -m644 arch/$KARCH/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch == */$KARCH/ || $arch == */arm/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  # Fix permissions
  chmod -R u=rwX,go=rX "$builddir"

  echo "Stripping build tools..."
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

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in ${pkgname[@]}; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
