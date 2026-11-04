================================================================================
XIAOMI / MIUI / HYPEROS DATA-COLLECTION DOMAIN REFERENCE
Companion notes for the block-list files
================================================================================

Companion rule files : xiaomi-privacy.hosts   (hosts format)
                       xiaomi-privacy.txt     (AdGuard syntax)
Document date        : 2026-11-05
Format               : plain text (ASCII layout, no Markdown)
Scope                : MIUI 13/14, HyperOS 1/2/3 (China and Global variants)

--------------------------------------------------------------------------------
CONTENTS
--------------------------------------------------------------------------------

  0.  What this document is (and is not)
  1.  The three interfaces that actually transmit an app list
      1.1  flash.sec.miui.com / flash.sec.intl.miui.com  (AntiDefraud)
      1.2  fraud-detection.engine.miui.com               (HyperOS engine)
      1.3  dm.wo.com.cn:18080/register                   (SystemHelper)
  2.  Full domain list with per-entry notes
      Group 1  App list / anti-fraud reporting
      Group 2  Package installer
      Group 3  Cloud scan engines
      Group 4  Security centre, other endpoints
      Group 5  Device fingerprint / usage statistics / telemetry
      Group 6  Advertising attribution
      Group 7  Mi Browser
      Group 8  Account / push / cloud services
      Group 9  Shared and carrier domains (disabled by default)
  3.  Choosing between the two rule files
  4.  Where DNS filtering still leaks
  5.  Recommended implementation layers (L1 to L4)
  6.  Explicitly unverified claims
  7.  Source list
  8.  Disclaimer and operational warnings


--------------------------------------------------------------------------------
0.  WHAT THIS DOCUMENT IS (AND IS NOT)
--------------------------------------------------------------------------------

The list is organised by WHAT IS COLLECTED, not by WHERE IT GOES. Three reasons:

  - The same data point (for example "which apps are installed") is collected
    by at least three independent components and sent to different endpoints.
  - Several domains carry both telemetry and a legitimate service function.
    Blocking them breaks normal features.
  - At the network layer you can only observe "a connection to domain X".
    You cannot see the payload. Therefore every description of collected
    content below carries an evidence grade. Anything without packet-capture
    or reverse-engineering support is labelled as inference.

EVIDENCE GRADES

  Grade A  Public packet capture, reverse engineering, or confirmation from a
           person associated with the vendor.

  Grade B  Present in multiple independently maintained block-lists; purpose
           reasonably inferable from the domain name and the component it
           belongs to; no per-domain capture evidence.

  Grade C  Inference, or a single non-authoritative source. Reference only.


--------------------------------------------------------------------------------
1.  THE THREE INTERFACES THAT ACTUALLY TRANSMIT AN APP LIST
--------------------------------------------------------------------------------

Everything else in the list is either adjacent to these three, or ordinary
telemetry unrelated to the app inventory.

1.1  flash.sec.miui.com  /  flash.sec.intl.miui.com   (AntiDefraud)

  Endpoint          https://flash.sec.miui.com/detect/app
  Global variant    https://flash.sec.intl.miui.com/detect/app
  Component         com.miui.guardprovider (MIUI security components),
                    com.miui.securitycenter
  Collected         Installed third-party app list, package names, versions,
                    signature digests, app display names, business
                    identifiers such as biz_id "virus_scan", the GuardProvider
                    UUID, and the MIUI version string.
  Trigger events    Launching Security > Settings; running a security scan;
                    checking for definition updates (regardless of whether
                    online definitions are enabled); force-stopping
                    SecurityCenter; clearing Security app data.
  Evidence          Grade A

  Sources
    - MinaMichita reverse-engineered MIUI 14 in February 2023 and released the
      Xposed module AntiAntiDefraud. Its README documents the intercepted
      payload shape:
        {"timestamp":"...","os":"...","biz_id":"virus_scan","uuid":"...",
         "content":[]}
    - Igor Eisberg, lead developer of MIUI EU, stated publicly on the Xiaomi
      European Community forum on 2023-02-05 that the AntiDefraud component is
      part of Mi Engine, that the app list is not sent to Tencent but to an API
      on a Xiaomi-owned server at https://flash.sec.miui.com/detect/app, and
      that the only difference between China and Global ROMs is the URL used.
    - MIUI Wiki (maintained by gledos) documents the incident and timeline.

  One caveat worth recording: in the captured samples the "content" field was
  frequently an empty array. That may mean the list is only populated after a
  match, or that the capture was truncated. A single capture is therefore not
  sufficient to conclude that no list is transmitted.

1.2  fraud-detection.engine.miui.com   (HyperOS fraud engine)

  Endpoint          https://fraud-detection.engine.miui.com/hyperos/fraud/app/report
  Component         System Security Component (separate component on HyperOS 3)
  Collected         Third-party app inventory (primary), plus scan results and
                    impersonating-app reports.
  Evidence          Grade A

  Source
    Packet analysis published 2025-10-31 by researcher MeT (@MeTerminator666).
    On a Xiaomi 14 running HyperOS 3, the System Security Component and
    SystemHelper each reported the app list, to this endpoint and to
    flash.sec.miui.com/detect/app respectively.

  The same analysis reports capability beyond the app list. The System
  Security Component performs fraud screening of phone, QQ and WeChat voice
  calls; it captures screen video at 720x1280, 24 fps, fixed portrait
  dimensions, recording roughly three seconds to a private cache file,
  reading the whole MP4, converting to Base64, encrypting and uploading to
  Xiaomi cloud, then waiting for a "risk or not" verdict. On a miss it deletes
  the fragment, waits two seconds and repeats; on a hit it retains a segment
  for user feedback. It also performs cloud detection of transcribed text and
  video fragments, and uploads raw audio when the user submits feedback.

  DNS filtering blocks the transport. It does not prevent local recording.

1.3  dm.wo.com.cn:18080/register   (SystemHelper, China Unicom side)

  Endpoint          http://dm.wo.com.cn:18080/register   (non-standard port)
  Component         com.mobiletools.systemhelper (SystemHelper)
  Collected         Apps used within the past 24 hours, full installed app
                    list, usage statistics and traffic figures. The serialised
                    payload includes IMEI, IMSI and ICCID fields.
  Behaviour detail  Skips entries whose package name begins with com.android;
                    focuses on third-party apps.
  Evidence          Grade A (same HyperOS 3 analysis as 1.2)
  Special risk      This hostname is shared with China Unicom's own services.
                    Blocking the domain at DNS level can break carrier
                    functions. Disabled by default in the supplied rule files.

  The IMEI / IMSI / ICCID detail is the most consequential part of this entry.
  It means the app inventory can be joined to SIM identity and hardware
  identity, producing a complete "this device, this SIM, these apps" profile.


--------------------------------------------------------------------------------
2.  FULL DOMAIN LIST WITH PER-ENTRY NOTES
--------------------------------------------------------------------------------

GROUP 1  APP LIST / ANTI-FRAUD REPORTING   (Grade A, core)

  flash.sec.miui.com
      App list, package names, versions, signature digests, GuardProvider
      UUID, MIUI version. Component: GuardProvider / Security Center.
  flash.sec.intl.miui.com
      Same behaviour, Global ROM variant.
  fraud-detection.engine.miui.com
      App list, scan results, impersonating-app reports. Component: System
      Security Component (HyperOS).

GROUP 2  PACKAGE INSTALLER   (Grade A)

  api.installer.xiaomi.com
      On install or update of an APK, the following fields are transmitted:
        apkSignature      APK signing certificate digest
        apkMd5            APK MD5
        localVersionCode  version code
        cpuArchitecture   supported ABIs
        client_uuid       installer component UUID
        device_type
        source            the app that triggered the install
        screen_size       screen resolution
        model / device name
      A geo-coordinate field was also discussed in the original thread, but it
      was not fully shown in the screenshot. Grade C for that field only.
      Evidence: V2EX packet-capture thread, 2022-01-26.

  preview-api.installer.xiaomi.com
      Pre-install information lookup and preview. Same general purpose.
      Grade B.

  Related but not a network behaviour: com.miui.packageinstaller once shipped an
  embedded blacklist.json that blocked installation by matching package name
  plus APK MD5, faking a "problem parsing the package" error on a hit. DNS
  filtering cannot affect this. It is listed here because it belongs to the same
  component's behavioural profile.

  A further item that remains unconfirmed: an email screenshot signed "Xiaomi
  Information Security and Privacy Committee" claimed that under the
  Anti-Telecom and Online Fraud Law, package names must be reported regardless
  of whether the user has enabled security checks. Grade C. Single screenshot,
  no second source. If cited, cite it as unverified.

GROUP 3  CLOUD SCAN ENGINES   (Grade A/B)

  These handle virus-definition updates, rule distribution and scan-verdict
  reporting. The collection target here is file and APK characteristics, not
  the installed-app inventory, although the two overlap.

  miav-cse.avlyun.com           Tencent / Antiy AVL cloud scan, primary.  Grade A
  miui-fxcse.avlyun.com         Antiy cloud scan, f-series endpoint.    Grade A
  ixav-cse.avlyun.com           Antiy cloud scan, international series. Grade B
  sdkconf.avlyun.com            Antiy SDK configuration delivery.       Grade B
  update.avlyun.sec.miui.com    Virus definition and rule updates.      Grade A
  logupdate.avlyun.sec.miui.com Scan log upload.                         Grade A
  log.avlyun.sec.intl.miui.com  International scan log upload.           Grade A
  virus-engine.sec.intl.miui.com  International virus engine endpoint.
                                  Newly observed during HyperOS 2.       Grade B
  tmfsdk.m.qq.com               Tencent Mobile Security SDK endpoint.    Grade B
  etl-xlmc-ssl.sandai.net       Xunlei / sandbox related reporting; appears
                                in community Xiaomi privacy lists.        Grade B

  Important distinction: the cloud-scan path collects file and APK
  characteristics; AntiDefraud collects the installed-app inventory. Different
  mechanisms, overlapping inference value. On Global ROMs the Tencent engine is
  disabled, but AntiDefraud is still present.

GROUP 4  SECURITY CENTRE, OTHER ENDPOINTS (*.sec.miui.com)   (Grade B)

  This group is mixed. Some endpoints are security checks, some are allow-list
  delivery, some are device and behaviour telemetry, some are ad attribution.
  Domains that resolve to *.sec.miui.com are not uniformly "app list".

  api.sec.miui.com               Security centre main API, includes status
                                 reporting.
  data.sec.miui.com              Security data reporting.
  adv.sec.miui.com               Promotional slot data inside Security app.
  adv.sec.intl.miui.com          International variant of the above.
  mazu.sec.miui.com              Security centre analytics. "mazu" is an
                                 internal Xiaomi telemetry system codename.
  xlmc.sec.miui.com              Security log / ETL reporting.
  etl.xlmc.sec.miui.com          ETL sub-path of the above.
  auth.be.sec.miui.com           Security component authentication and device
                                 registration.
  authbe.sec.intl.miui.com       International variant.
  api.tw06.xlmc.sec.miui.com     Log reporting shard.
  adinfo.ra1.xlmc.sec.miui.com   Advertising information reporting.
  r.sg.sec.miui.com              Regional node. Grade C.
  api.miui.security.xiaomi.com   Legacy security centre API domain.

  On the "xlmc" prefix: the community has associated it with virus-definition
  and scan activity. Separately, an IP address 157.148.47.204 has been
  described in community posts as a "Guangdong anti-fraud centre" server. That
  claim is Grade C. It is not recommended to trust it or to block the IP:
  single-IP attribution is unreliable, and IP blocking affects unrelated
  services sharing the same range.

GROUP 5  DEVICE FINGERPRINT / USAGE STATISTICS / TELEMETRY   (Grade A/B)

  data.mistat.xiaomi.com         MiStat primary reporting endpoint: device
                                 identifiers, model, OS version, event
                                 telemetry. Grade A
  data.mistat.intl.xiaomi.com    International MiStat endpoint. Grade A
  abtest.mistat.xiaomi.com       A/B experiment assignment and exposure. B
  storeconfig.mistat.intl.xiaomi.com  Statistics SDK configuration. B
  tracking.miui.com              System-level behaviour tracking. A
  tracking.intl.miui.com         International variant. A
  fcanr.tracking.miui.com        Tracking sub-path. B
  a0.app.xiaomi.com              Xiaomi tracking API; listed in community
                                 privacy lists. B
  metok.sys.miui.com             System metrics and statistics. B
  geofence.sys.miui.com          Geofencing. Location-adjacent. B
  cc.sys.miui.com                System configuration and channel statistics.
  ccc.sys.miui.com               Variant of the above.
  cc.sys.intl.xiaomi.com         International variant.
  ccc.sys.intl.xiaomi.com        International variant.
  privacy.api.intl.miui.com      International privacy endpoint; newly
                                 observed during HyperOS 2. B
  mcc-intl.inf.miui.com          International information-collection endpoint;
                                 newly observed during HyperOS 2. C
  api.collect.data.intl.miui.com Literally named "collect data". B
  api.device.xiaomi.net          Device registration and identifiers. B
  userid.xiaomi.com              User and device ID association. B
  idm.api.io.mi.com              Mi Home / IoT device and account linkage. B
  api.io.mi.com                  Mi Home / IoT core API. B
  stat.xiaomi.com                General statistics.
  a.stat.xiaomi.com              General statistics, alternate shard.
  api.d.xiaomi.com               Data API.
  d.xiaomi.com                   Data endpoint.

GROUP 6  ADVERTISING ATTRIBUTION   (Grade A/B)

  ad.xiaomi.com                  Advertiser endpoint: device identifier plus
                                 ad slot.
  ad.mi.com                      Domestic ad endpoint.
  ad.intl.xiaomi.com             International ad endpoint.
  api.ad.xiaomi.com              Ad requests and impression callbacks.
  api.ad.intl.xiaomi.com         International variant.
  new.api.ad.xiaomi.com          Newer ad API generation.
  globalapi.ad.xiaomi.com        Global ad API.
  e.ad.xiaomi.com                Ad event reporting.
  cdn.ad.xiaomi.com              Ad creative delivery, with distribution
                                 statistics.
  ssp.ad.xiaomi.com              Supply-side platform.
  ssp.xiaomi.com                 Supply-side platform, legacy domain.
  zeus.ad.xiaomi.com             Ad delivery engine.
  zeus.ad.intl.xiaomi.com        International ad delivery engine.
  sdkconfig.ad.xiaomi.com        Ad SDK configuration.
  sdkconfig.ad.intl.xiaomi.com   International ad SDK configuration.
  diagnosis.ad.xiaomi.com        Ad pipeline diagnostics, includes device and
                                 network environment.
  diagnosis.ad.intl.xiaomi.com   International variant.
  log.ad.xiaomi.com              Ad logging.
  track.ad.xiaomi.com            Click and conversion tracking.
  m.track.ad.xiaomi.com          Mobile tracking sub-path.
  t.track.ad.xiaomi.com          Tracking sub-path.
  clock.ad.xiaomi.com            Ad timing calibration.

  The risk in this group is identity stitching. Device identifiers collected by
  ad SDKs can be joined with the data in Groups 1 and 5, binding an advertising
  profile to an app inventory on the same physical device.

GROUP 7  MI BROWSER   (Grade A/B)

  api.browser.miui.com           Browser main API. The path
                                 /bsr/update/blackurl delivers a URL block-list
                                 to
                                 /data/data/com.android.browser/files/data/cacblacklist/blacklist.json
  security.browser.miui.com      "Safe URL detection". Submits the URL being
                                 visited to a server for a verdict.
  httpdns.browser.miui.com       HttpDNS. Exposes DNS resolution requests.
  r.browser.miui.com             Home-page recommendations.
  nav.browser.miui.com           Navigation and trending content.
  hd.browser.miui.com            Content delivery for the above.
  activity.browser.intl.miui.com International campaign and telemetry.

  The blackurl item has a clear origin. Discovered in April 2018, the list
  content at that time was entirely political-news related, and only mainland
  China devices downloaded it. A Hong Kong journalist testing a Hong Kong
  handset did not find the block-list.

GROUP 8  ACCOUNT / PUSH / CLOUD SERVICES   (Grade B)

  From a sensitivity standpoint this group collects the most personal data:
  contacts, call records, file inventories. However these are part of cloud
  services the user explicitly enables. Keep this group if the goal is only to
  reduce passive collection; accept functional loss if the goal is full
  disconnection.

  register.xmpush.xiaomi.com     Push registration, device token.
  api.xmpush.xiaomi.com          Push API.
  file.xmpush.xiaomi.com         Push file channel.
  sandbox.xmpush.xiaomi.com      Push sandbox environment.
  resolver.msg.xiaomi.net        Message routing resolution.
  solve.msg.xiaomi.net           Message routing resolution, alternate.
  api.account.xiaomi.com         Account authentication, device binding.
  ruac.account.xiaomi.com        Account subsystem.
  api.micloud.xiaomi.net         Mi Cloud sync.
  pdc.micloud.xiaomi.net         Photo and data cloud.
  find.api.micloud.xiaomi.net    Find Device.
  contactapi.micloud.xiaomi.net  Contacts.
  phonecallapi.micloud.xiaomi.net  Call records.
  sfsapi.micloud.xiaomi.net      File service.
  statusapi.micloud.xiaomi.net   Service status.
  cn.app.chat.xiaomi.net         Xiaomi messaging, China.
  app.chat.xiaomi.net            Xiaomi messaging.
  api.chat.xiaomi.net            Messaging API.

  Blocking consequences: push stops entirely, sync and cloud services fail,
  Find Device stops working.

GROUP 9  SHARED AND CARRIER DOMAINS   (disabled by default)

  dm.wo.com.cn
      See section 1.3. Shared with China Unicom's own services. High collateral
      risk at DNS level. A safer approach is per-app network control, blocking
      only com.mobiletools.systemhelper.

  ssl-cdn.static.browser.mi-img.com
      Static resource CDN with distribution statistics attached. Blocking
      breaks image, theme and wallpaper loading.

  sec-cdn.static.xiaomi.net
      Security component static resources. Same caveat.

  api.jr.mi.com
      Xiaomi Finance and wallet interface, also carries telemetry. Blocking
      affects wallet and finance features.


--------------------------------------------------------------------------------
3.  CHOOSING BETWEEN THE TWO RULE FILES
--------------------------------------------------------------------------------

  xiaomi-privacy.hosts
      Format        hosts (0.0.0.0 domain)
      Matching      exact hostname
      Strength      will not over-block sibling subdomains; easy to audit
                    entry by entry
      Weakness      newly introduced subdomains will be missed
      Best for      RethinkDNS, AdAway, dnsmasq, Pi-hole

  xiaomi-privacy.txt
      Format        AdGuard syntax (||domain^)
      Matching      domain suffix wildcard
      Strength      covers new subdomains; low maintenance
      Weakness      wildcards over-block, especially whole-domain blocks such
                    as ||avlyun.com^
      Best for      AdGuard App and AdGuard Home, Surge, Clash (converted)

  Recommendation: start with the hosts file. It is controllable. Watch DNS logs
  for a week, then promote confirmed-safe high-frequency domains to wildcard
  rules and move up to the AdGuard file.


--------------------------------------------------------------------------------
4.  WHERE DNS FILTERING STILL LEAKS
--------------------------------------------------------------------------------

  1.  New domains keep appearing. Domains newly observed during HyperOS 2
      include virus-engine.sec.intl.miui.com, privacy.api.intl.miui.com,
      mcc-intl.inf.miui.com, find.api.micloud.xiaomi.net,
      sdkconfig.eu.xiaomi.com, eu.global.market.xiaomi.com,
      eu-gamecenter.api.intl.miui.com and theme.cdn.intl.miui.com. Any static
      list lags by definition.

  2.  Data reroutes. Users have reported that after blocking network access for
      the app store and the security centre individually, install and uninstall
      information still left the device through other internet-capable Xiaomi
      components, including Xiao Ai, AI call features, wallpaper components and
      maintenance-mode components. Managing which components may reach the
      network is stronger than managing domain names.

  3.  Hard-coded IPs and non-standard ports. dm.wo.com.cn:18080 can be blocked
      by hostname, but if a component switches to direct IP connections the
      block fails. This requires IP or CIDR rule support, or per-app network
      control.

  4.  Nothing above TLS is auditable. DNS only shows which domain was contacted,
      not what was sent. Verification requires MITM (PCAPdroid with a custom CA,
      noting that system components often use certificate pinning) or direct
      reverse engineering.

  5.  Local collection is not upload. The screen recording and voice
      transcription described in 1.2 are captured locally before transmission.
      DNS filtering only severs the second half.


--------------------------------------------------------------------------------
5.  RECOMMENDED IMPLEMENTATION LAYERS
--------------------------------------------------------------------------------

  L1  Private DNS pointing at NextDNS or a RethinkDNS custom slot, with this
      list imported.
      Blocks: all domain-level reporting.
      Does not block: direct-IP connections, embedded DoH.

  L2  L1 plus per-app rules denying network access to
      com.miui.securitycenter, com.mobiletools.systemhelper and
      com.miui.packageinstaller.
      Blocks: all channels from those components, including unknown new
      domains.
      Does not block: reporting originating inside the system framework.

  L3  L2 plus pm disable-user via Shizuku or maintenance mode to disable
      SystemHelper and parts of the security components.
      Blocks: component level. Most thorough.
      Caveat: HyperOS restricts direct disabling of system apps via ordinary
      ADB.

  L4  L3 plus the Xposed module AntiAntiDefraud, which puts GuardProvider into
      debug mode, blocks at source and logs the payload.
      Blocks: at source, and provides verification.
      Requires: LSPosed.

  The value of AntiAntiDefraud is not only blocking but verification. Once
  enabled, LSPosed logs show records shaped like
    Intercept={"timestamp":"...","biz_id":"virus_scan","uuid":"...","content":[]}
  which answers directly the question of what was actually transmitted.


--------------------------------------------------------------------------------
6.  EXPLICITLY UNVERIFIED CLAIMS
--------------------------------------------------------------------------------

  The following statements were not corroborated by a reliable second source.
  Retain the qualifier whenever citing them.

  1.  "The data flows into the national anti-fraud centre."
      The reporting endpoints and trigger conditions on the Xiaomi side are
      established. There is no publicly verifiable evidence chain showing which
      institution's system ultimately receives the data. The original community
      analysis by @LoopDNS explicitly stated that it had not determined that
      the component uploads to a national anti-fraud data centre.

  2.  "IP 157.148.47.204 belongs to a Guangdong anti-fraud centre."
      Single source, no corroboration. Do not block the IP on this basis.

  3.  "api.installer.xiaomi.com uploads geo-coordinates."
      The original screenshot did not fully display that field. The author used
      hedged wording. Grade C.

  4.  The "Xiaomi Information Security and Privacy Committee" email screenshot.
      One image, no second source. Grade C.

  5.  Field-level payload contents for most domains.
      Only Groups 1 and 2 have capture or reverse-engineering support. The rest
      are inferred from naming and component. Grade B. Actual fields require
      your own capture to confirm.

  6.  Differences across system versions and regions.
      China, Global and EU ROMs behave differently. For example, the Tencent
      engine is disabled on Global ROMs. This list targets China ROM behaviour
      primarily and was not verified per version.


--------------------------------------------------------------------------------
7.  SOURCE LIST
--------------------------------------------------------------------------------

  S1   github.com/MinaMichita/AntiAntiDefraud and mirrors
       MIUI 14 AntiDefraud behaviour, trigger conditions, payload structure,
       Xposed interception approach.

  S2   Igor Eisberg (MIUI EU lead developer), public statement on the Xiaomi
       European Community forum, 2023-02-05.
       Confirms flash.sec.miui.com/detect/app as the app-list endpoint; confirms
       the Tencent engine is disabled on international ROMs; confirms the Global
       URL difference.

  S3   MIUI Wiki, maintained by gledos (ggame.gledos.science).
       AntiDefraud timeline, installer upload fields, blackurl incident,
       bootloader and system restriction history, ROM variant differences.

  S4   Researcher MeT (@MeTerminator666), packet analysis published
       2025-10-31.
       HyperOS 3 endpoint fraud-detection.engine.miui.com/hyperos/fraud/app/report;
       SystemHelper to dm.wo.com.cn:18080/register; screen recording and voice
       collection implementation.

  S5   V2EX thread "MIUI 13 packet capture: install uploads coordinates and
       package names", 2022-01-26.
       Field list for api.installer.xiaomi.com.

  S6   github.com/LoopDns/Fuck-you-MIUI (MIhosts / Fhosts).
       Cloud-scan blocking baseline, and the original caveat that data flow to
       an anti-fraud centre was not established.

  S7   github.com/unknownFalleN/xiaomi-dns-blocklist, including issue #44.
       Telemetry and advertising domain baseline; new HyperOS 2 domains.

  S8   ACL4SSR/ACL4SSR, MIUIPrivacy.list fragment.
       Privacy domain compilation, derived from S6.

  S9   "MIUI: block ads and detection hosts", Zishi forum thread.
       Practical block-list for ad, browser and security domains.

  S10  "How to block cloud scanning and installed-app upload on Chinese
       phones", Yixueshe.
       Cloud-scan component attribution (Tencent Antiy AVL), block list.

  S11  "Taking back device control, part 2: blocking iQOO/OriginOS third-party
       install cloud checks", Lihvarya.
       Cross-verification: OPPO and vivo also use viav-cse.avlyun.com; Antiy is
       shared across multiple vendors.

  S12  Check Point Research, 2019, GuardProvider vulnerability report.
       Confirms GuardProvider integrates Avast, AVL and Tencent engines.

  S13  Lithuanian National Cyber Security Centre report 2021; Taiwan NCC
       report 2022; XDA analysis by Adam Conway.
       MiAdBlacklistConfig incident, establishing that the system does contain
       content-filtering list mechanisms.


--------------------------------------------------------------------------------
8.  DISCLAIMER AND OPERATIONAL WARNINGS
--------------------------------------------------------------------------------

  This list is intended for personal privacy reduction on your own devices. It
  does not constitute an allegation of unlawful conduct by any vendor. Any
  compliance determination regarding these domains must come from a regulatory
  authority.

  Blocking has functional costs. Cloud scanning stops working. Third-party APK
  installation may be blocked or may repeatedly prompt a risk warning. Push,
  sync and Find Device may malfunction. Enable Groups 5 through 8 first, confirm
  normal operation, and only then enable Groups 1 through 3.

  If the device is used for payments, government services or corporate work,
  do not block Groups 3 and 4. Those relate to security verification, and
  blocking them can trigger risk controls.

  Domains and behaviour change with system versions. This list reflects public
  material available up to November 2026. For ongoing coverage, defer to the
  upstream lists named in the header of xiaomi-privacy.hosts.

================================================================================
END OF DOCUMENT
================================================================================
