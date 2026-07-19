# sing-box URLTest freeze fix

This directory records the temporary core patch deployed while upstream
SagerNet/sing-box pull request #4256 is still open.

- Base version: `v1.13.14`
- Base revision: `25a600db24f7680ad9806ce5427bd0ab8afe1114`
- Patch revision: `219dbcc7bd0676301d725a2958e1bc7b7c7a57af`
- Local version: `1.13.14-urltestfix.4256-naive-musl`
- Go version: `1.25.12`
- Cronet build revision: `98d539ce67568fb911654e66a14cf4247ed833ec`
- NaiveProxy revision: `888e114241c89b05fac4e4ee01482d7bd89ca15a`
- Target: OpenWrt 23.05.5, `x86_64-openwrt-linux-musl`

The patch adds a read deadline to each URL test, propagates batch
cancellation, caps a health-check batch at twice the TCP timeout, and removes
stale results for probes that do not finish.

## Build

Apply `4256-v1.13.14.patch` to a clean `v1.13.14` checkout. Naive requires
Cronet and cannot be added to the generic `CGO_ENABLED=0` build by only adding
a tag. Prepare the exact OpenWrt musl toolchain used by `cronet-go` first:

```sh
git clone --branch v1.13.14 --depth 1 https://github.com/SagerNet/sing-box.git
cd sing-box
git apply ../4256-v1.13.14.patch

git init ../cronet-go
git -C ../cronet-go remote add origin https://github.com/SagerNet/cronet-go.git
git -C ../cronet-go fetch --depth=1 origin 98d539ce67568fb911654e66a14cf4247ed833ec
git -C ../cronet-go checkout FETCH_HEAD
git -C ../cronet-go submodule update --init --recursive --depth=1
(cd ../cronet-go && go run ./cmd/build-naive --target=linux/amd64 --libc=musl download-toolchain)

toolchain="$PWD/../cronet-go/naiveproxy/src"
export CGO_LDFLAGS='-fuse-ld=lld'
export CC="$toolchain/third_party/llvm-build/Release+Asserts/bin/clang --target=x86_64-openwrt-linux-musl --sysroot=$toolchain/out/sysroot-build/openwrt/23.05.5/x86_64"
export CXX="$toolchain/third_party/llvm-build/Release+Asserts/bin/clang++ --target=x86_64-openwrt-linux-musl --sysroot=$toolchain/out/sysroot-build/openwrt/23.05.5/x86_64"

tags='with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_ccm,with_ocm,with_naive_outbound,badlinkname,tfogo_checklinkname0,with_musl'
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build \
  -trimpath \
  -tags "$tags" \
  -ldflags "-X github.com/sagernet/sing-box/constant.Version=1.13.14-urltestfix.4256-naive-musl -X internal/godebug.defaultGODEBUG=multipathtcp=0 -checklinkname=0 -s -w -buildid=" \
  -o sing-box-1.13.14-urltestfix.4256-naive-openwrt-amd64 \
  ./cmd/sing-box
```

Before installation, run the candidate against the exact Momo configuration:

```sh
./sing-box-1.13.14-urltestfix.4256-naive-openwrt-amd64 check -c config.json
```

The deployed stable binary has SHA-256
`d40c6239245abcb07eeea8382879af6740c908e9dcdf9000a18d8997fb273b6f`.
The previous binary is retained on the test router as
`/usr/bin/sing-box.pre-naive-20260719-1440` for rollback. The separately built
`v1.14.0-alpha.47` candidate has SHA-256
`43225c497f498aad2c787d89d5540b67ea7201e6b0396b5f745e8c03c9b28557`;
it passed configuration and isolated Naive tests but is not deployed because it
is a prerelease.

An OpenWrt package upgrade can overwrite `/usr/bin/sing-box`. Remove this
temporary patch after upgrading to an upstream release that includes #4256.
