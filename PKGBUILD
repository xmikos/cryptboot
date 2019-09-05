# Maintainer: Michal Krenek (Mikos) <m.krenek@gmail.com>
pkgname=cryptboot
pkgver=1.1.0
pkgrel=1
pkgdesc="Encrypted boot partition manager with UEFI Secure Boot support"
arch=('any')
url="https://github.com/xmikos/cryptboot"
license=('GPL3')
depends=('cryptsetup' 'grub' 'efibootmgr' 'efitools' 'sbsigntools')
install="cryptboot.install"
source=("${pkgname}-${pkgver}.tar.gz::$url/archive/v$pkgver.tar.gz")

package() {
  cd "$srcdir/$pkgname-$pkgver"
  install -Dm755 cryptboot "$pkgdir/usr/bin/cryptboot"
  install -Dm755 cryptboot-efikeys "$pkgdir/usr/bin/cryptboot-efikeys"
  install -Dm755 cryptboot-grub-warning "$pkgdir/etc/cryptboot-grub-warning"
  install -Dm644 cryptboot.conf "$pkgdir/etc/cryptboot.conf"
  mkdir -p "$pkgdir/usr/local/bin/"
  ln -s "/etc/cryptboot-grub-warning" "$pkgdir/usr/local/bin/grub-install"
}

# vim:set ts=2 sw=2 et:
