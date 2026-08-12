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

Break-glass accounts remain excluded throughout (`CA-BreakGlassAccounts - Exclude` — the ONLY
exclusion, per the locked PTS model).

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
