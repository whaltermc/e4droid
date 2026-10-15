# e4droid

**e4mc, patched to run on Android / PojavLauncher.**

This directory is an add-on patchset for the e4mc Minecraft mod. It keeps the
upstream `link.e4mc` namespace completely intact and adds all Android-specific
code under `dev.whaltermc.e4droid`.

The GitHub fork is [whaltermc/e4droid](https://github.com/whaltermc/e4droid),
forked from `vgskye/e4mc-minecraft-architectury`. The repository keeps its
upstream name for the code, but the fork itself is renamed to `e4droid`.

## Why patches instead of a hard fork?

e4mc is MIT licensed and lightly maintained; a full rename (mod id, packages,
maven coordinates, i18n keys) would make rebasing onto upstream painful and
would break e4mc's own relay / WireGuard-less design (hosts run e4mc, vanilla
clients just connect to the domain). So we:

- leave `link.e4mc.*`, `e4mc` mod id, `assets/e4mc_minecraft/*` and the
  `link.e4mc:e4mc_minecraft-*` maven coordinates alone, and
- drop Android behaviour into `dev.whaltermc.e4droid.*` via small, reviewable
  git patches.

## What the patches do

| # | Patch | Change |
|---|-------|--------|
| 1 | `0001-add-android-compat-and-natives-loader.patch` | Adds `dev.whaltermc.e4droid.AndroidCompat` (Android/ABI detection) and `AndroidNatives` (loads bionic builds of quiche + iroh). |
| 2 | `0002-quiclime-android-nio-and-native-load.patch` | `QuiclimeSession`: pre-loads Android natives, forces the Netty **NIO** datagram channel (PojavLauncher has no epoll/kqueue natives), and fails fast with a clear message if no bionic quiche is present. |
| 3 | `0003-dialtone-android-guard.patch` | `DialtoneAmbientSession`: disables Dialtone with a clear exception when no bionic iroh is present, instead of crashing. The exception is caught by `DialtoneChannel` and fails only that connection. |

## Apply

From the repository root:

```sh
git am dev.whaltermc.e4droid/patches/*.patch     # keeps authorship / creates commits
# or, without committing:
git apply dev.whaltermc.e4droid/patches/*.patch
```

Reverse with `git am -3 --abort` / `git apply -R ...` or `git checkout -- common`.

Then build as usual (`./gradlew build`) and drop into PojavLauncher's mods folder.

## Android natives

Upstream ships glibc-linked Linux natives. Those cannot be `dlopen()`ed by
PojavLauncher's bionic JVM, so e4droid loads Android builds itself, first:

```
<mods>/e4droid-natives/<abi>/libnetty_quiche.so    # required for QUIClime
<mods>/e4droid-natives/<abi>/libiroh_java.so       # optional, for Dialtone
```

`<abi>` is one of `arm64-v8a`, `armeabi-v7a`, `x86_64`, `x86`, chosen
automatically from `os.arch`. You can override the location with the JVM
property `-De4droid.nativesDir=/path/to/natives`.

These natives must be built against bionic (Android NDK):
- quiche / netty-incubator-codec-quic, see
  [vgskye/netty-incubator-codec-quic](https://github.com/vgskye/netty-incubator-codec-quic)
- iroh-java, see [vgskye/iroh-java](https://github.com/vgskye/iroh-java)

Until an Android build of one of them is present, e4droid degrades gracefully:
- without **quiche**: QUIClime hosting is disabled and `/e4mc doctor` reports it;
- without **iroh**: Dialtone is disabled, but normal relay hosting still works.

## Testing off-device

Set `-De4droid.forceAndroid=true` to exercise the Android code paths on desktop.

## License

Same as upstream, [MIT](../LICENSE).
