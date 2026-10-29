<p align="center"><img src="docs/brand/splitin-logo.png" alt="SplitIn" width="96" height="96"></p>

# Group Share

Private app links with an invite list, network restrictions and an expiry date.
`@splitin/group-share` is a Node.js library, CLI and reverse-proxy gateway. It creates
subdomains for upstream applications and checks access before forwarding requests.

Use it for controlled previews, internal demos and office/VPN-only tools. For example,
share a preview with three named colleagues for seven days, restricted to the office
network. Sending the URL through AirDrop does not grant access by itself.

## Status and distribution

This is the canonical standalone source repository. The source package is version
`0.1.0`, with ESM, CommonJS, TypeScript declarations and a `group-share` executable.
The migration does not establish that this version is available on npm. Use the source
or a locally packed artifact until a registry release is verified.

The bundled CLI is a development starting point: its email-code delivery callback only
logs a request notice; it does **not** send the code. Configure a real `deliverCode`
adapter when embedding the gateway. File storage and in-process challenges are local
implementations; shared production persistence and identity delivery need operator work.

## How access works

Every request, including requests with an existing session cookie, must pass all three checks:

1. The share exists, has not expired and has not been revoked.
2. The authenticated email is on the share group's invite list.
3. The connection IP belongs to a CIDR range allowed for that group.

Empty email or network lists deny access. A forwarded URL does not bypass these checks.
There is no password-only or anyone-with-the-link mode. Protect the upstream application's
own origin as well: this gateway only controls traffic that passes through it.

The admin page runs on the parent host. It asks **Who**, **Where** (upstream URL), and
**How long** (1, 7 or 30 days), then opens the device share sheet or offers a copy link.
The simple flow uses the `default` group, updates its membership and seeds its allowed
network from the operator IP when needed. Advanced controls expose other groups, CIDRs
and revocation. Recipients use the generated subdomain and email-code sign-in.

## Quick start from source

Requires Node.js 20 or later; CI checks Node 22 and 24.

```sh
git clone https://github.com/splitintech/group-share.git
cd group-share
npm ci
npm test
npm run build
npm run smoke
node examples/local-preview/preview.mjs
```

Open `http://127.0.0.1:8788/`; the example operator token is `dev-token`.
This preview uses local fixtures, not production identity delivery or wildcard DNS.
See [local preview](examples/local-preview/README.md).

To produce an installable artifact after building:

```sh
npm pack
# In a separate consumer project:
npm install /absolute/path/splitin-group-share-0.1.0.tgz
```

## Library API

```ts
import {
  mintShare, MemoryGroupStore, MemoryShareStore,
} from '@splitin/group-share/server';

const groups = new MemoryGroupStore();
const shares = new MemoryShareStore();
await groups.put({ groupId: 'design', members: [], cidrs: [] });
await groups.addMember('design', 'ada@example.com');
await groups.addCidr('design', '203.0.113.0/24');
const share = await mintShare({ groups, shares }, {
  groupId: 'design',
  originUrl: 'http://127.0.0.1:4010',
});
```

The example network is a documentation range; use your actual permitted network.

| Entry point | Purpose |
| --- | --- |
| `@splitin/group-share` | Core access, group and share APIs |
| `@splitin/group-share/server` | Server integration, gateway and stores |
| `@splitin/group-share/identity` | Identity provider interfaces and adapters |
| `group-share` | CLI executable after installation |

This package uses Node APIs. Run the gateway and identity code on the server.

## CLI and configuration

After building, use `node dist/cli.js`; an installed package provides `group-share`.

```sh
export GROUP_SHARE_SECRET="$(openssl rand -hex 32)"
export GROUP_SHARE_BASE_DOMAIN=share.example.test
export GROUP_SHARE_SECURE=0  # local HTTP only
node dist/cli.js groups add design ada@example.com
node dist/cli.js allow design 127.0.0.1/32
node dist/cli.js share --group design --origin http://127.0.0.1:4010
node dist/cli.js serve
# Revoke a previously generated label:
# node dist/cli.js revoke <label>
```

The CLI server's code delivery remains a stub; the example is not a complete recipient-login deployment.

| Setting | Purpose |
| --- | --- |
| `GROUP_SHARE_SECRET` | Required session signing key |
| `GROUP_SHARE_OPERATOR_TOKEN` | Admin token; defaults to the signing key |
| `GROUP_SHARE_BASE_DOMAIN` | Parent domain for generated labels |
| `GROUP_SHARE_DATA_DIR` | Local group/share storage directory |
| `GROUP_SHARE_DEFAULT_CIDRS` | Initial allowed network ranges |
| `GROUP_SHARE_TRUSTED_PROXIES` | Peers permitted to supply forwarded client IPs |
| `GROUP_SHARE_DIRECTORY` | Path to a JSON email directory |
| `GROUP_SHARE_PORT`, `GROUP_SHARE_HOST` | Listener settings |
| `GROUP_SHARE_SECURE` | Set `0` only for local HTTP; production uses secure cookies |

## Deployment and operations

Provide a wildcard DNS record and TLS certificate for `*.share.example.com`, plus the
parent admin host. Configure trusted proxy ranges explicitly. Production cookies use
`Secure`, `HttpOnly`, `SameSite=Lax` and the `__Host-` prefix. Supply real email delivery
and a suitable store implementation before expecting production recipient access.

See [operator guide](docs/operator.md), [threat model](docs/threat-model.md) and
[security policy](SECURITY.md). Revoking a share disables its gateway link; it does not
remove the upstream app or replace that app's authorization.

## Repository and checks

- `src/core`: access decisions, CIDRs, sessions and share lifecycle.
- `src/gateway`, `src/admin`: proxy and administrative handlers.
- `src/identity`, `src/store`: identity and persistence interfaces/implementations.
- `src/ui`, `ui`: admin and recipient pages/assets.
- `tests`, `examples`: behavioral tests and local demonstrations.

Run `npm test`, `npm run build`, `npm run smoke` and `npm run check:packed`.
The packed-consumer check installs the actual tarball into a fresh project and checks
its public imports and CLI. Publishing stays disabled until repository-specific npm
trusted publishing is configured and a release is deliberately enabled.

## Contributing and provenance

See [CONTRIBUTING.md](CONTRIBUTING.md), [migration provenance](docs/migration.md),
[validation record](docs/migration/validation.json), and [release setup](docs/repository-release.md).
Relevant Git history, source inventories, original licenses and artwork are retained.
The migration does not publish packages, configure credentials or deploy infrastructure.

## License

MIT. See [LICENSE](LICENSE).
