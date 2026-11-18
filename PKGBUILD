# Maintainer: sukkergris
pkgname=bluos-tui-git
pkgver=r2.f9b8e4b
pkgrel=1
pkgdesc='Terminal client for BluOS players (unofficial)'
arch=(any)
url='https://github.com/sukkergris/bluos-tui'
license=(MIT)
depends=(bash fzf curl ffmpeg gvfs-smb iproute2)
makedepends=(git)
provides=(bluos-tui)
conflicts=(bluos-tui)
source=("git+$url.git")
sha256sums=(SKIP)

pkgver() {
    cd bluos-tui
    printf 'r%s.%s' "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

package() {
    cd bluos-tui
    install -Dm755 bin/bluos "$pkgdir/usr/bin/bluos"
    install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
    install -Dm644 README.md "$pkgdir/usr/share/doc/$pkgname/README.md"
}
