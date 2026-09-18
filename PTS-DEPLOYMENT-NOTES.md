# PTS Deployment Notes — CA005 / CA006 (App Protection enforcement)

> PTS-specific operational notes for this fork. The full PTS CA docs (Baseline Standard,
> Deployment Package) live in Hudu / the PTS workspace — this file carries only what must sit
> next to the policies.

## ⚠️ CA005/CA006 deployment order — do NOT skip (mobile-email lockout risk)

These two policies enforce Intune App Protection (MAM) on iOS/Android. Sequenced wrong, they
block users out of mobile email.

1. Deploy the **iOS App Protection policy** (ONE of the three PTS tracks from
   `pts-intune-baseline` → `IOS/AppProtection/`) and confirm it's applying.
2. Ensure users have **Microsoft Authenticator** (required broker app — devices must register
   in Entra to satisfy the grant) **and** the managed apps (Outlook, Teams).
3. ⚠️ **Users on the native iOS Mail app will be blocked** the moment this enables — native
   Mail cannot satisfy an app protection grant. Identify and migrate them first.
4. Deploy the CA policy **report-only**, watch ~1 week, identify stragglers.
5. Migrate stragglers → enable.

Break-glass accounts remain excluded throughout (`CA-BreakGlassAccounts - Exclude`, per the
locked PTS model — see the exclusion-model section below).

## Exclusion model — break-glass only, with ONE documented constant

Every PTS CA policy excludes `CA-BreakGlassAccounts - Exclude` and nothing else, **except
CA000**, which additionally excludes the **Directory Synchronization Accounts** role
(`d29b2b05-8046-44ba-8758-1e26182fcf32`). This is Microsoft's standard guidance: requiring MFA
on the Entra Connect / cloud-sync service account breaks directory synchronisation. It is a
role-scoped exclusion (not a user or group), it is intentional, and it is **not** a precedent
for other exclusions — any additional exclusion request is a baseline-owner decision.

**Time-boxed travel exclusions (owner decision 2026-09-17).** Travellers are handled per person
and per trip, never by widening the tenant: **CIPP Vacation Mode** on CA001 (CIPP creates and
reuses a `Vacation Exclusion - CA001-…` group on that policy and schedules the traveller in on
the start date and out on the end date), a **CIPP Travel Policy** for the destination (a temporary
block-everywhere-except-the-destination policy and named location that CIPP removes when the trip
ends), and a **Huntress Managed ITDR Expected rule** on the traveller's identity with the same
dates — see Hudu doc 10, Decision 5. The `Vacation Exclusion - <policy>` groups CIPP creates are
the only other exclusion a PTS CA policy may carry: never populated by hand, never on any policy
but CA001. The earlier rule (temporarily adding the destination country to `ALLOWED COUNTRIES`)
is retired — it opened the country to every user in the tenant and relied on a calendar entry for
its removal. SIGIL shows the exclusion on the CA001 row with the traveller and the end date, and
flags one that outlives its date.

**Shared-device accounts (owner decision 2026-09-18).** Room boards, kiosks, shop-floor terminals and
signage sign in with an always-on interactive session on a device visitors sit in front of. They are the
`SharedDevices` persona: group `PTS-Shared-Device-Accounts` (dynamic on the `shd-` UPN prefix), templates
**CA500** (block any sign-in from a device not stamped `extensionAttribute1 = PTS-SharedDevice`; the
device class rides `extensionAttribute2`) and **CA501** (require compliant device, held until the
device-management layer exists). No MFA policy: Microsoft's Teams Rooms guidance says interactive MFA is
unsupported or not to be enforced for these accounts, sign-in frequency and persistent browser are
unsupported, and device code flow must stay open on Android. Because the Global 0xx set targets all
users, `PTS-Shared-Device-Accounts` is the **second documented constant exclusion**, on CA000, CA003,
CA004, CA005 and both CA006 variants — exactly Microsoft's guidance to exclude resource accounts from
every existing policy. CA500 deploys report-only; stamp every board before it goes On, because an
unregistered device matches a negative filter and is blocked.

## Machine-readable manifest — `Config/pts-baseline.json`

The Graph exports carry the settings; `Config/pts-baseline.json` carries what they cannot: per policy,
the persona, the interference tier (`low` / `known` / `high`), the blast radius in words, the layers it
depends on, the soak (`day` / `week` / `hold` / `none`), the doc 05 §C4 enable step, whether report-only
exists, and a client gate. SIGIL seeds it alongside the templates and drives the board, the playbook and
the "still blocked?" reading from it. Tiers under rule H: everything still goes report-only first — a
`low` policy soaks one business day, `known` a week with its blast-radius checklist, `hold` until every
layer it depends on reads present, and the two continuous-access-evaluation policies deploy Off until
their step. Edit the manifest and tag the repo; never the templates, for these facts.

## Licensing note (corrects an old claim)

CA005/CA006 were previously held back as "not licensed" — **that was wrong.** Business Premium
includes **Intune Plan 1**, which includes App Protection Policies. MAM was always licensed;
the real blocker was that PTS hadn't deployed an Intune baseline. That baseline now exists
(`pts-intune-baseline`), so CA005/CA006 are **deployable** for Intune-managed tenants.

## Grant-control status (post 2026-06-30 retirement)

Microsoft retired `Require approved client app` (`approvedApplication`) on 2026-06-30.
Swept 2026-08-12: **no policy in this fork carries the retired grant** — upstream migrated
CA005 to `compliantApplication` (Require app protection policy) before the cutoff, and
CA006-RequireAppProtection already used it. Any future policy must use `compliantApplication`
only.
