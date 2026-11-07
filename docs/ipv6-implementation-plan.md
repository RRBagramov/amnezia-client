# IPv6 support: audit and implementation plan

Goal: connect to a server whose IPv4 address is unreachable from the client
(blocked by the access provider) over the server's public IPv6 address, with
user IPv4 traffic carried inside the tunnel. Secondary goals: IPv6 traffic
inside the tunnel and self-hosted management over IPv6.

Related upstream items: issue 1037 (installer accepts only IPv4, IPv6 config
import broken on Linux), PR 3364 (brackets for IPv6 in the UAPI endpoint; open,
not merged; superseded here by routing the address to the IPv6 field, which the
daemons already bracket), issues 3375/3376 (Windows site split tunnelling, not
related to the transport).

## Audit

### Endpoint path for WireGuard / AmneziaWG (desktop)

```
UI / import -> hostName in the protocol config
  -> WireGuardProtocol::start   (NetworkUtilities::getIPAddress resolves names)
  -> LocalSocketController::activate (JSON for the daemon)
  -> Daemon::parseConfig -> InterfaceConfig
  -> WireguardUtils{Macos,Linux,Windows}::updatePeer (UAPI "endpoint=")
     WireguardUtilsWindows::addInterface -> InterfaceConfig::toWgConf (wg-quick)
  -> UDP socket in amneziawg-go / wireguard-nt
```

Broken before this change:

| Place | Problem |
|---|---|
| `client/mozilla/localsocketcontroller.cpp` | server address always sent as `serverIpv4AddrIn`, so an IPv6 literal produced `endpoint=2001:db8::10:51820` |
| `client/daemon/interfaceconfig.cpp` (`toWgConf`, used on Windows) | `Endpoint = host:port` without brackets |
| `client/platforms/windows/daemon/windowsdaemon.cpp`, `NetworkUtilities::AdapterIndexTo` | adapter lookup via IPv4-only `GetBestRoute` |
| `client/platforms/ios/WGConfig.swift` | wg-quick text `Endpoint = host:port` without brackets, rejected by WireGuardKit |
| `client/server_scripts/{wireguard,awg,awg_legacy}/template.conf` | native config export `Endpoint = $SERVER_IP_ADDRESS:port` without brackets |
| `client/core/controllers/selfhosted/importController.cpp` | `QUrl::fromUserInput` for `Endpoint`; unbracketed IPv6 without a port was rejected |
| `client/mozilla/localsocketcontroller.cpp` | every AllowedIPs/split-tunnel range sent with `isIpv6:false`, bare IPv6 hosts as `/32`; `Address = v4, v6` sent as one IPv4 string |

Already correct: UAPI writers in the three daemons bracket the IPv6 field;
Android (`InetEndpoint`); daemon exclusion routes for IPv6 on macOS/Linux
route monitors; Windows firewall allows an IPv6 server.

### Kill switch

| Platform | Problem |
|---|---|
| macOS (pf, last match wins) | `250.blockIPv6` blocks the IPv6 server even when it is in `<allownets>` |
| Linux (iptables, first match) | `110.allowNets` is IPv4-only and sits after `250.blockIPv6` in the IPv6 chain |
| Windows (WFP) | IPv6 server is allowed; no change |

### Self-hosted

- `InstallUiController::setProcessedServerCredentials` split `host:port` on
  every `:`, so any IPv6 literal broke. libssh 0.11 accepts IPv6 literals in
  `SSH_OPTIONS_HOST` (no brackets), so only the parsing had to change.
- `PayloadSender::parseEndpoint` used `lastIndexOf(':')` and kept brackets.
- Server containers publish ports with `docker run -p port:port/udp`; Docker's
  userland proxy also listens on the host IPv6 address and forwards to the
  IPv4 container, so a dual-stack server is reachable over IPv6 without
  script changes (per Docker documentation; not verified on a server).
- Server scripts are IPv4-only inside (NAT, docker networks, image builds,
  `ifconfig eth0:0` in OpenVPN, mtproxy ipify). This matters only for an
  IPv6-only server and is out of scope here.

### Name resolution

`NetworkUtilities::getIPAddress` returns the first address from
`QHostInfo::fromName`, which keeps the `getaddrinfo` order (RFC 6724 with
`AI_ADDRCONFIG`). There is no hard IPv4 preference; the system policy decides,
including NAT64-synthesised AAAA on IPv6-only networks. Names are resolved on
each connect, so a reconnect picks up new addresses. iOS and Android resolve in
WireGuardKit / `parseInetAddress`.

### Other protocols

- OpenVPN: `remote <host> <port>` with `proto udp|tcp`; OpenVPN 2.4+ picks the
  family from the resolved address. Kill switch allow-list now covers IPv6.
- XRay (desktop): blocked. `XrayProtocol::setupRouting` calls
  `StopRoutingIpv6`, which disables IPv6 system-wide on Linux and routes IPv6
  to loopback on Windows, cutting the connection to an IPv6 server. A fix
  needs per-server IPv6 exclusion routes in the service router; not done.
- IPsec/IKEv2, Cloak, Shadowsocks: not audited in depth.

## Implementation stages

1. Shared parsing: `NetworkUtilities::parseHostPort`, `endpointHost`,
   `endpoint` (Qt `QHostAddress`/`QUrl`, no regex; never splits an
   unbracketed IPv6; validates port 1..65535). Used by the SSH credentials
   input, `PayloadSender`, WireGuard import, wg-quick writer, template vars.
2. WG/AWG transport: IPv6 address goes to `serverIpv6AddrIn`; wg-quick and iOS
   bracket IPv6; Windows adapter lookup via `GetBestRoute2`; native export
   templates use `$SERVER_ENDPOINT_HOST`.
3. Kill switch: macOS `allownets` rule is `quick`; Linux `110.allowNets` is
   installed for both families before `250.blockIPv6` and fills
   `ip6tables` for IPv6 entries; the service kill switch toggles it for both.
4. Tunnel ranges: IPv6 ranges marked `isIpv6:true` with `/128` default;
   `Address = v4, v6` split into device IPv4/IPv6 addresses; the ULA
   placeholder stays when the config has no IPv6 address.
5. Tests: `test_host_port_parsing`, `test_wireguard_endpoint_import`.

Not done (follow-ups):

- IPv6 inside the tunnel with the kill switch on (macOS/Linux): the kill
  switch keeps blocking all IPv6 except the server (fail-closed).
- XRay IPv6 endpoint on desktop (see above).
- IPv6-only servers: server scripts and Docker networks.
- DNS servers given as IPv6 in imported configs (`DNS =` regex is IPv4-only).
- UI hint text on the credentials page still shows an IPv4 example.

## Verification

- `test_host_port_parsing` built against Qt and run on macOS.
- Remaining checks need the full Conan build: `ctest` for the import test,
  then manual connects on macOS: IPv4 literal, IPv6 literal, AAAA-only name,
  A+AAAA name, kill switch on/off; adding a self-hosted server as
  `[2001:db8::10]:22`.
