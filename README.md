# Network Fabric Studio

A browser-based network planner, topology editor and **Arista EOS IPv4 configuration draft generator**. It has three independent working areas plus a local list of named project copies. It does not connect to or deploy configurations to switches.

## Try it without installing anything

Open `network-fabric-studio.html` in a modern desktop browser. The portable edition contains the application, styles and configuration engine in one file and does not require a server. Downloads and browser storage were checked in Chromium. Use **Export project** to keep a JSON backup; browser storage belongs to the browser and origin/file location and is not cloud synchronization.

An HTML preview inside a chat client may restrict scripts, downloads or storage. If it does, download the file and open it in your browser. This deliverable is not a published or registered ChatGPT embedded app.

## Run the source project

Unzip the project and enter the `network-canvas` directory. Use Node.js 24 and npm 11; this is the combination tested for this delivery.

```sh
npm ci
npm run dev -- --host 127.0.0.1
```

Open `http://127.0.0.1:8080`. The existing developer startup script also works from the project directory: `sh startup.sh`. It uses port 8080 and defaults to all interfaces; set `FABRIC_DEV_HOST=127.0.0.1` for loopback-only use.

No database, API key or sign-in is needed for the network planner. The inherited authentication, database and hosting scaffolding remains in the source. Database migration intentionally skips when `DATABASE_URL` is absent; the planner uses browser storage.

```sh
npm run typecheck
npm test
npm run lint
npm run build
npm run preview
```

The built preview uses `http://127.0.0.1:8081`. The existing Linux helper `npm run preview:restart` can manage it where `/proc` is available. Stop it with `npm run preview:stop`.

Create the single-file edition:

```sh
npm run build:portable
```

Output: `dist-portable/network-fabric-studio.html`. This uses the same UI and domain code as the main application.

Regenerate the reviewed example projects and their containerlab topologies:

```sh
npm run examples
```

## Browser regression checks

```sh
npx playwright install chromium
npm run test:browser
npm run test:browser:built
node scripts/fabric-browser.mjs --portable
```

Build the normal and portable outputs first. Each server-based test starts and stops its own local server; keep its port free. The test writes screenshots and a JSON verdict under `artifacts/browser`. Set `FABRIC_QA_OUTPUT` to preserve separate runs. In Linux CI, `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH` can select an already installed compatible Chromium; the test uses the included `@sparticuz/chromium` launch arguments when this override is supplied. Ordinary use needs only Playwright's browser.

## The three workflows

1. **Plan Addressing First:** choose Layer 2 or Layer 3 and a project protocol; choose counts for routers, spines, leaves, member leaves and hosts. Create an explicit suggested plan, then edit the inventory, connections, VLANs, gateways, ASNs, loopbacks and MLAG rows. Live repair hints identify the affected field and correction. At generation you can optionally fill only blank cells, missing rows and missing links; nonblank values are preserved even when invalid.
2. **Build Topology Manually:** enter a name and model. Add devices with the toolbar or drag them onto the canvas. Drag a connection dot to another device, or select a device, choose a connection type and click Connect. Select, double-click or right-click devices and links to edit them. The link editor contains both endpoint addresses, interfaces, routing protocol, VLANs and LACP fields. MLAG pair creation adds a two-member peer bundle and a row in Addressing. The canvas has zoom/reset controls and Shift-drag panning. Optional device model, EOS release and maximum Ethernet port fields add traceability and interface-capacity checking.
3. **Use Recommended Defaults:** select the single-fabric Layer 2 MLAG/VARP example, two-fabric L2LS example, routed BGP example or routed OSPF example. The reference buttons replace the reference workspace only. Your manual topology and independent addressing plan remain saved separately.

Each device has a **Fabric / pod** field. Devices with the same value are evaluated as one fabric. Suggested spine-to-leaf and host connections stay within that fabric. This allows several independent two-spine L2LS fabrics in one project without treating their combined spine count as one L3LS fabric. Automatic and manual checks require links between different fabrics to be routed.

The live topology breakdown classifies each fabric from its actual spine-to-leaf links, MLAG pair and SVI placement. It also shows the Layer 3 edge. Generated results include `topology-breakdown.md` and `bgp-breakdown.md`; the BGP document lists the actual underlay neighbors, ASNs, router IDs, BGP states, a simplified best-path order and a verification sequence.

The address policy is explicit: routed point-to-point endpoints use private `10.0.0.0/8` addresses with `/30`; SVI and host subnets use `172.16.0.0/16`; MLAG peer endpoints use `192.168.0.0/16` with `/30`. Missing prefixes, wrong masks, addresses outside the assigned pool, network/broadcast endpoints, subnet mismatch and overlap produce actionable messages.

An MLAG peer bundle always uses `Ethernet1` and `Ethernet2` on both switches. Creating MLAG moves existing connections on those ports to the next available interfaces; removing the peer connection removes the pair and releases both ports. Imported projects with the earlier single-link representation are upgraded during validation. The canvas draws the peer bundle as two parallel orange links with a vertical bundle marker. Dual-homed MLAG client links use orange trunk paths and a larger client-bundle marker.

**Back/Forward** follows page history and disables unavailable directions. **Undo/Redo** follows project-change history (also Ctrl/Cmd+Z and Ctrl/Cmd+Shift+Z) and disables unavailable actions. A continuous device drag is one undoable change. **Save copy** adds the current project to the named browser-local project list; **Open** and Delete manage those copies. **Export project** saves editable JSON. **Import project** validates its full structure before replacement. **Download selected file** downloads an individual device configuration, host worksheet or report. **Export all configurations** creates one text bundle with labeled file sections; it is not a deployable concatenated switch configuration.

## Configuration scope

Supported generation: IPv4 L2 access and trunks, VLANs, SVIs, VARP, LACP, MLAG, routed eBGP underlay and single-area OSPF area 0. Multiple L2LS fabrics may share one project; each requires exactly two MLAG spines, spine gateway SVIs and Layer 2 spine-to-leaf trunks. L3LS requires routed spine-to-leaf links, leaf gateway SVIs and no direct spine-to-spine link. Use unique eBGP ASNs except for members of an explicitly defined MLAG pair. Routed links require explicit `/30` addresses at both ends. Hosts receive OS-neutral addressing worksheets rather than switch CLI. Validation also reports disconnected routed islands and single-uplink leaf failure points.

Unsupported intent is either blocked where modeled or listed as out of scope: VXLAN/EVPN overlays, VRFs, IPv6, static routes, routing-policy design, redistribution, multi-area OSPF, general iBGP route reflection, NAT, device management/access setup and non-EOS vendors. A role named Router still uses the EOS configuration target.

Passing the built-in plan checks means the implemented checks found no blocking errors. It does not certify every possible topology or an EOS release. Validate the output on your target virtual or physical switches before deployment. See `VERIFICATION.md` for the actual test results and remaining work.

Each of the four example directories contains a `containerlab.yml` topology that loads the generated cEOS startup configurations. See `lab/README.md`. These lab files were generated and inspected, but not executed here because no licensed cEOS image, Docker runtime or target EOS release was available.

## Where the code lives

| Location                                | Responsibility                                                                   |
| --------------------------------------- | -------------------------------------------------------------------------------- |
| `src/routes`                            | Starter, planning, design and review page URLs                                   |
| `src/components/planning`               | Inventory, addressing tables and connection planning                             |
| `src/components/topology`               | SVG canvas and device/connection editors                                         |
| `src/components/review`                 | Validation results, config previews and downloads                                |
| `src/lib/fabric/store.ts`               | Independent workspace state and browser persistence                              |
| `src/lib/fabric/profiles.ts`            | Explicit reference inventories and address plans                                 |
| `src/lib/fabric/fill-missing.ts`        | Non-destructive completion of blank fields, rows and connections                 |
| `src/lib/fabric/topology-analysis.ts`   | Per-fabric classification, Layer 3 edge and BGP breakdown reports                |
| `src/lib/fabric/validate-project.ts`    | Generation blockers and consistency checks                                       |
| `src/lib/fabric/generator.ts`           | Deterministic EOS drafts and host worksheets                                     |
| `src/lib/fabric/project-file.ts`        | JSON import schema and export helpers                                            |
| `src/portable.tsx`                      | Offline entry point sharing the existing components                              |
| `scripts`                               | Existing platform tooling plus browser regression and portable packaging scripts |
| `lab` and `examples/*/containerlab.yml` | cEOS/containerlab verification path                                              |

The ZIP also contains the untouched original upload for recovery and verification evidence from this repair.
