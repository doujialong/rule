# sing-box URLTest freeze fix

This directory records the temporary core patch deployed while upstream
SagerNet/sing-box pull request #4256 is still open.

- Base version: `v1.13.14`
- Base revision: `25a600db24f7680ad9806ce5427bd0ab8afe1114`
- Patch revision: `219dbcc7bd0676301d725a2958e1bc7b7c7a57af`
- Local version: `1.13.14-urltestfix.4256`
- Go version: `1.24.7`

The patch adds a read deadline to each URL test, propagates batch
cancellation, caps a health-check batch at twice the TCP timeout, and removes
stale results for probes that do not finish.

## Build

Apply `4256-v1.13.14.patch` to a clean `v1.13.14` checkout, then build the
static Linux amd64 binary with the tags used by the active Momo configuration:

```sh
git clone --branch v1.13.14 --depth 1 https://github.com/SagerNet/sing-box.git
cd sing-box
git apply ../4256-v1.13.14.patch

tags='with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_ccm,with_ocm,badlinkname,tfogo_checklinkname0'
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -trimpath \
  -tags "$tags" \
  -ldflags "-X github.com/sagernet/sing-box/constant.Version=1.13.14-urltestfix.4256 -X internal/godebug.defaultGODEBUG=multipathtcp=0 -checklinkname=0 -s -w -buildid=" \
  -o sing-box-1.13.14-urltestfix.4256-linux-amd64 \
  ./cmd/sing-box
```

Before installation, run the candidate against the exact Momo configuration:

```sh
./sing-box-1.13.14-urltestfix.4256-linux-amd64 check -c config.json
```

An OpenWrt package upgrade can overwrite `/usr/bin/sing-box`. Remove this
temporary patch after upgrading to an upstream release that includes #4256.
