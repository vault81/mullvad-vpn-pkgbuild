# Maintainer: Mark Wagie <mark dot wagie at tutanota dot com>
# Contributor: Matthew McGinn <mamcgi at gmail dot com>
# Contributor: alicewww <almw at protonmail dot com>
# Contributor: David Birks <david at tellus dot space>
# Contributor: Jeff Henson <jeff at henson dot io>
# Contributor: Linus Färnstrand <linus at mullvad dot net>
# Contributor: Emīls Piņķis <emil at mullvad dot net>
# Contributor: Andrej Mihajlov <and at mullvad dot net>
pkgname=mullvad-vpn-tailscale-fix-git
_pkgname=mullvad-vpn
provides=('mullvad-vpn')
conflicts=('mullvad-vpn')
pkgver=2023.4_ts_fx
pkgrel=1
pkgdesc="The Mullvad VPN client app for desktop"
arch=('x86_64')
url="https://www.mullvad.net"
license=('GPL3')
depends=('gtk3' 'iputils' 'libnotify' 'nss')
makedepends=('cargo' 'git' 'go' 'libxcrypt-compat' 'nodejs>=16' 'npm>=8.3' 'protobuf')
options=('!lto')
install="$_pkgname.install"
_tag=573ded0eab7526d65c83948c7351a3d00be58390  # tags/2023.4^0
_commit=29a4c7205e78c651fcd1b8c3a55181c0d86a50d3
source=("git+https://github.com/vault81/mullvadvpn-app.git#commit=${_tag}?signed"
        "git+https://github.com/mullvad/mullvadvpn-app-binaries.git#commit=${_commit}?signed"
        'no-rpm.diff'
        "$_pkgname.sh")
sha256sums=('SKIP'
            'SKIP'
            'ea35edffea2cbbb05586abce19581fdd9f133801ed47e6af30fa64a29c5cf116'
            '2262346cb57deb187fe32a88ccd873dab669598889269088e749197c6e88954f')

pkgver() {
  cd "$srcdir/mullvadvpn-app"
  git describe --tags | sed 's/-/./g'
}

prepare() {
  cd "$srcdir/mullvadvpn-app"
  git submodule init
  git config submodule.dist-assets/binaries.url "$srcdir/mullvadvpn-app-binaries"
  git -c protocol.file.allow=always submodule update

  # Disable building of rpm
  patch --strip=1 gui/tasks/distribution.js < ../no-rpm.diff

  export CARGO_HOME="$srcdir/cargo-home"
  export RUSTUP_TOOLCHAIN=stable
  cargo fetch --locked --target "$CARCH-unknown-linux-gnu"

  pushd wireguard/libwg
  export GOPATH="$srcdir/gopath"
  mkdir -p "../../build/lib/$CARCH-unknown-linux-gnu"
  go mod download -x
  popd

  pushd gui
  echo "Installing JavaScript dependencies..."
  export npm_config_cache="$srcdir/npm_cache"
  npm ci
  popd
}

build() {
  cd "$srcdir/mullvadvpn-app"
  export CARGO_HOME="$srcdir/cargo-home"
  export RUSTUP_TOOLCHAIN=stable
  local RUSTC_VERSION=$(rustc --version)
  local PRODUCT_VERSION=$(cargo run -q --bin mullvad-version)

  echo "Building Mullvad VPN ${PRODUCT_VERSION}..."

  echo "Building wireguard-go..."
  pushd wireguard/libwg
  export GOPATH="$srcdir/gopath"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export CGO_LDFLAGS="${LDFLAGS}"
  export GOFLAGS="-buildmode=pie -trimpath -ldflags=-linkmode=external -mod=readonly -modcacherw"
  go build -v -o "../../build/lib/$CARCH-unknown-linux-gnu"/libwg.a -buildmode c-archive
  popd

  # Clean module cache for makepkg -C
  go clean -modcache

  echo "Building Rust code in release mode using ${RUSTC_VERSION}..."
  export CARGO_TARGET_DIR=target
  cargo build --frozen --release

  echo "Preparing for packaging Mullvad VPN ${PRODUCT_VERSION}..."
  mkdir -p build/shell-completions
  for sh in bash zsh fish; do
    echo "Generating shell completion script for ${sh}..."
    cargo run --bin mullvad --frozen --release -- shell-completions ${sh} \
      build/shell-completions/
  done

  echo "Updating relays.json..."
  cargo run --bin relay_list --frozen --release > dist-assets/relays.json

  # Move binaries to correct locations in dist-assets
  binaries=(
    mullvad-daemon
    mullvad
    mullvad-problem-report
    libtalpid_openvpn_plugin.so
    mullvad-setup
    mullvad-exclude
  )
  for binary in ${binaries[*]}; do
    cp target/release/${binary} dist-assets/${binary}
  done

  # Build Electron GUI
  pushd gui
  echo "Packing Mullvad VPN ${PRODUCT_VERSION} artifact(s)..."
  export npm_config_cache="$srcdir/npm_cache"
  npm run pack:linux --release
  popd
}

package() {
  cd "$srcdir/mullvadvpn-app"

  # Install main files
  install -d "$pkgdir/opt/Mullvad VPN"
  cp -r dist/linux-unpacked/* "$pkgdir/opt/Mullvad VPN/"

  chmod 4755 "$pkgdir/opt/Mullvad VPN/chrome-sandbox"

  # Install services
  install -Dm644 dist-assets/linux/mullvad{-daemon,-early-boot-blocking}.service -t \
    "$pkgdir/usr/lib/systemd/system/"

  # Install binaries
  install -Dm755 dist-assets/{mullvad,mullvad{-daemon,-exclude}} -t "$pkgdir/usr/bin/"

  # Link to the problem report binary
  ln -s "/opt/Mullvad VPN/resources/mullvad-problem-report" "$pkgdir/usr/bin/"

  # Link to the GUI binary
  install -m755 "$srcdir/$_pkgname.sh" "$pkgdir/usr/bin/$_pkgname"

  # Install completions
  install -Dm644 build/shell-completions/mullvad.bash \
    "$pkgdir/usr/share/bash-completion/completions/mullvad"
  install -Dm644 build/shell-completions/_mullvad -t \
    "$pkgdir/usr/share/zsh/site-functions/"
  install -Dm644 build/shell-completions/mullvad.fish -t \
    "$pkgdir/usr/share/fish/vendor_completions.d/"

  # Install desktop file & icons from deb
  cd dist
  ar x *.deb
  bsdtar -xf data.tar.xz
  install -Dm644 "usr/share/applications/$_pkgname.desktop" -t \
    "$pkgdir/usr/share/applications/"

  for icon_size in 16 32 48 64 128 256 512 1024; do
    icons_dir=usr/share/icons/hicolor/${icon_size}x${icon_size}/apps
    install -Dm644 ${icons_dir}/$_pkgname.png -t "$pkgdir/${icons_dir}/"
  done
}
