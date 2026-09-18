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
