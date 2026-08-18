# ConnectivityService: preserve system-default UID rules

This fork contains a targeted LineageOS 23.2 fix for per-app VPN default
network handling.

## The bug

LineageOS adds transport UID allowlists to `ConnectivityService`. When a
per-app VPN default changes, `makeDefaultForApps()` passed that app-specific
network to `updateUidDefaultNetworkRules()` as if it were the global system
default.

That could rewrite the allowlist for the real Wi-Fi or cellular default,
demoting its default priority and leaving unrelated allowed UIDs without the
expected route. The failure could look like connectivity loss or traffic
being sent through the wrong network when a per-app VPN/no-service default was
changed.

## The fix

The patch removes exactly one invalid call from:

```text
service/src/com/android/server/ConnectivityService.java
ConnectivityService.makeDefaultForApps(...)
```

Global default rule updates remain in `makeDefaultNetwork()`, where the
argument really is the system default. Per-app UID changes continue through
the existing `modifyNetworkUidRanges()` path.

No firewall, routing daemon, VPN implementation or persistent state is added.

## Compatibility

This branch targets:

- LineageOS 23.2 / Android 16;
- `packages/modules/Connectivity` base
  `c9f3e7795256bd4a3f99d02e604e24fd1552c2b3`;
- the Android 16 Connectivity module layout used by that branch.

It is Android framework source, not an APK, Magisk module or KernelSU module.
Do not copy the repository to a phone and do not install its JAR directly.

## Build usage

Use this repository at `packages/modules/Connectivity` in a complete
LineageOS 23.2 source tree:

```xml
<project name="DurkaEbanaya/android_packages_modules_Connectivity"
    path="packages/modules/Connectivity"
    remote="github"
    revision="f50ce3d2015c8d1ef2df476b35d2082372813120" />
```

Build the complete matching system/APEX output and install it as part of a
consistently signed OTA. A framework JAR transplanted from another build can
have incompatible signatures, resources, APIs or module metadata.

## Regression test

The patch adds:

```text
testVpnDefaultDoesNotChangeSystemDefaultAllowlist()
```

The test establishes a physical system-default network with the default
allowlist priority, changes a per-app VPN default, and verifies that the
physical network is neither removed nor rewritten with the non-default
priority.

The focused source test was reviewed and added, but was not executed locally
because a complete Android build/`atest` environment was unavailable. A
runtime-equivalent bytecode hotfix was exercised on the target LineageOS 23.2
phone during normal per-app VPN use.

## Related review

[LineageOS Gerrit 496806](https://review.lineageos.org/c/496806)

## Safety

Back up data before installing a custom framework build. Keep a known-good
boot/system/vendor/vbmeta rollback set and do not mix framework artifacts from
different Android or LineageOS revisions.
