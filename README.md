# Netkit Lab Generator

**Build and visualize Kathará network laboratories in your browser.**

Netkit Lab Generator provides a visual editor for network devices, interfaces, addressing, and routes. It generates Kathará laboratory files and offers an interactive topology preview. This repository is a fork of the [Kathará Framework's Netkit Lab Generator](https://github.com/KatharaFramework/Netkit-Lab-Generator), with additional IPv6, browser-library, and container-deployment features.

**Quick start:** use the prebuilt Docker image; no Node.js installation or local Docker build is required.

```sh
docker run -d --name netkit-lab-generator -p 8080:8080 \
  --restart unless-stopped ghcr.io/jadjay/netkit-lab-generator:latest
```

Open **http://localhost:8080**. To deploy using the repository's Compose file, see [Prebuilt image with Docker Compose](#prebuilt-image-with-docker-compose).

> **What the web container does not do:** It serves the editor and three official lab archives. It does not run Kathará machines or host ZIP files created by visitors. The generated laboratory must be run separately on a system with Kathará installed.

## Features

- **IPv4 and IPv6:** configure addresses, interface domains, gateways, static routes, and dual-stack laboratories, with input validation.
- **Topology preview:** visualize the laboratory's network graph in the browser.
- **Generate and download:** export a laboratory shell script (`.sh`) or ZIP archive (`.zip`), plus a reusable editor configuration (`.config`).
- **Save & Load:** import/export UTF-8 `.config` files, including accented metadata.
- **My saved laboratories:** save, list, reload, and delete editable snapshots using **IndexedDB** in the current browser profile.
- **Kathará example laboratories:** load editable representations of three upstream labs: **Two Computers**, **Basic IPv4**, and **Static Routing**. The conversion preserves supported interfaces, IP addresses, and active routes, but is **not lossless**; see [Example laboratories](#example-laboratories).
- **Server-aware curl preview:** derive the currently visited server's HTTP(S) origin and port for an example `/labs/<name>.zip` command. A ZIP must already exist on the server for the command to work.
- **Docker and GHCR:** run the static web application on an unprivileged Nginx container; published images are built with GitHub Actions.

![Topology graph](images/screencapture-201801143.png)

## Installation

### Prebuilt image with Docker Compose

Requirements: Docker Engine with the Compose plugin, and access to the GitHub Container Registry image.

Clone this repository and deploy **without a local image build**:

```sh
git clone https://github.com/jadjay/Netkit-Lab-Generator.git
cd Netkit-Lab-Generator
docker compose -f compose.ghcr.yaml up -d
curl -fI http://localhost:8080/
```

Then open **http://localhost:8080** in a browser.

The Compose service uses `ghcr.io/jadjay/netkit-lab-generator:latest` and publishes port `8080`.

Update an existing deployment:

```sh
docker compose -f compose.ghcr.yaml pull
docker compose -f compose.ghcr.yaml up -d
```

Stop the service:

```sh
docker compose -f compose.ghcr.yaml down
```

**Registry access:** the GHCR package may be private. If an anonymous pull is denied, authenticate with a GitHub account that has package access, or make the package public in GitHub **Packages → Package settings → Change visibility**. Publishing via GitHub Actions uses `GITHUB_TOKEN`; no personal token is required for the workflow.

### Build locally with Docker Compose

If you want to build from the checked-out source instead of pulling GHCR:

```sh
docker compose up -d --build
curl -fI http://localhost:8080/
```

The multistage `Dockerfile` uses an unprivileged Nginx Alpine image and downloads three native Kathará archives at a **pinned upstream revision**. Building requires network access to `raw.githubusercontent.com`; the build checks each ZIP with `unzip -t`.

Stop with `docker compose down`. If you use both Compose files, note that they expose the **same host port** and should not be run simultaneously without changing the port mapping.

### Use without Docker

The UI is a static web application. Serve the repository directory from any static HTTP server; for example, with Python 3:

```sh
python3 -m http.server 8080
```

Then open **http://localhost:8080**. You can also open `index.html` directly in a browser for basic use, but some browser APIs (including persistent storage and native file pickers) may be restricted on `file://`.

Alternatively, the original project supports an Electron desktop mode. Its development commands are defined in `package.json` (`npm start`, `npm run package`, `npm run make`); this mode is **not** part of the Docker image.

## Using the generator

1. Fill in the laboratory description and metadata.
2. Add machines, assign interfaces and collision domains, configure IPv4/IPv6 addressing and routing.
3. Open **Topology** to inspect the network.
4. Under **Download Lab**, download a generated `.sh` script or a `.zip` archive. Inspect the files before running them with Kathará.
5. Under **Save & Load**, export/import `.config` files or save snapshots to your browser library.

**Personal laboratory storage is browser-local.** IndexedDB data is not stored in the Docker container, shared across browsers, or automatically backed up. Export a `.config` file to keep a portable backup.

### Example laboratories

The **Save & Load → Kathará example laboratories** table includes three curated examples from [KatharaFramework/Kathara-Labs](https://github.com/KatharaFramework/Kathara-Labs):

| Example | What it demonstrates | Editable in the UI |
| --- | --- | --- |
| Two Computers | Two hosts on one collision domain | Yes |
| Basic IPv4 | Two routers, three PCs, IPv4 addresses and routes | Yes |
| Static Routing | Two routers and two PCs; routes to complete | Yes |

The editor's examples are **partial conversions** of upstream `lab.conf` and `.startup` content. Not every native Kathará feature fits the editor model: fixed MAC addresses, image choices, Wireshark services, and other specialized files may not be preserved. Commented-out exercise routes remain inactive. Use each example's **Source** link to inspect its complete upstream files.

The Docker image also serves the **original upstream ZIP archives**, separately from these editable examples.

## Download published lab ZIPs with curl

For an installed Docker image, three *native* Kathará lab ZIP files are available over HTTP:

```sh
curl -fL http://localhost:8080/labs/two-computers.zip -o two-computers.zip
curl -fL http://localhost:8080/labs/basic-ipv4.zip -o basic-ipv4.zip
curl -fL http://localhost:8080/labs/static-routing.zip -o static-routing.zip
unzip -t basic-ipv4.zip
```

The description-derived names are also available. For example:

```sh
curl -fL \
  http://localhost:8080/labs/basic_ipv4_configurations_ping_traceroute_and_arp.zip \
  -o basic-ipv4.zip
```

Using `curl -fL` is important: `-f` fails on HTTP 404 instead of saving an HTML error page under a `.zip` filename.

The **Download Lab → curl** preview uses the current page's `window.location.origin`, including a custom port or HTTPS reverse-proxy hostname. **This generates an example URL, not an upload.** A lab created or modified in the browser is **not** published to `/labs/`; its displayed curl command returns 404 until you separately host that ZIP. On `file://`, no HTTP origin exists, so a `YOUR_SERVER` placeholder is used.

## Architecture and limitations

| Component | Responsibility |
| --- | --- |
| Browser JavaScript / AngularJS | Editor, topology UI, file generation and downloads |
| IndexedDB | Per-browser personal lab snapshots |
| Nginx (Docker) | Static frontend and three upstream ZIP downloads under `/labs/` |
| Kathará on a separate machine | Runs the generated laboratories |

The container requires **no application database, Node.js runtime, Electron runtime, Kathará daemon, or mounted Docker socket**. It cannot execute laboratories on the host. Electron's **Run on Host** capability is not available in the web container.

For access from other computers, consider an **HTTPS reverse proxy** and appropriate access controls. Native browser file pickers may require a secure context (HTTPS or localhost); regular downloads remain available otherwise.

## Development and tests

This is a static frontend; **Node.js is only needed to run the regression suite** or work with the legacy Electron tooling, not to serve the Docker image.

```sh
node --test tests/*.test.cjs
```

The tests cover IPv4/IPv6 handling, generated files, UTF-8 imports, browser storage, example-lab conversion, Docker configuration, and GHCR workflow requirements. A passing structural test suite does not replace a Docker/HTTP smoke test or a real Kathará run.

For a Docker smoke test:

```sh
docker compose up -d --build
curl -fI http://localhost:8080/
curl -fL http://localhost:8080/labs/basic-ipv4.zip -o /tmp/basic-ipv4.zip
unzip -t /tmp/basic-ipv4.zip
```

### Image publishing

The [GitHub Actions workflow](.github/workflows/publish-image.yml) runs tests and a Docker build for pull requests **without publishing**. Pushes to `master` publish the `latest` and commit-SHA tags to `ghcr.io/jadjay/netkit-lab-generator`; version tags such as `v1.6.0` publish matching semantic-version tags. Publication uses the repository-scoped `GITHUB_TOKEN` with package-write permission.

## Upstream, credits, and license

This repository is a fork of [KatharaFramework/Netkit-Lab-Generator](https://github.com/KatharaFramework/Netkit-Lab-Generator). The original project and its contributors retain their respective credits; see the [upstream contributors](https://github.com/KatharaFramework/Netkit-Lab-Generator/graphs/contributors). The project metadata credits **Gaetano Bonofiglio and Lorenzo Ariemma**. Additional changes in this fork are available in its Git history and pull requests.

Useful links: [Kathará](https://www.kathara.org/) · [Kathara-Labs](https://github.com/KatharaFramework/Kathara-Labs) · [Netkit](http://wiki.netkit.org/index.php/Main_Page).

The project is distributed under the **GNU General Public License version 3 or later**. See [LICENSE](LICENSE).

### FAQ

**Can a machine be both a web server and a name server?** The original generator documentation explains that machines with the same name can share configuration; shared files are appended. Review the resulting output carefully.

**Why does my custom laboratory ZIP return HTTP 404?** A generated ZIP is downloaded to your browser, not uploaded to Nginx. Only the three bundled example ZIPs are available at the documented `/labs/` endpoints.
