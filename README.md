# Real-World React Native Apps

Large, actively maintained, open-source React Native apps that ship to the public app stores.

**Inclusion criteria:** source is public on GitHub, the app is published on the App Store and/or Google Play, and the repo had commits in the last 30 days.

_Data as of 2026-10-19._

## 1. Apps

| App | Category | Expo / Bare | RN | Expo SDK | New Arch | ⭐ Stars | License | App Store | Google Play | Other |
|---|---|---|---|---|---|---|---|---|---|---|
| [Joplin](https://github.com/laurent22/joplin/tree/dev/packages/app-mobile) | Notes | Bare + Expo modules | 0.86.3 | 57 | ✅ | 56.8k ¹ | Custom ² | [iOS](https://apps.apple.com/app/id1315599797) | [Android](https://play.google.com/store/apps/details?id=net.cozic.joplin) | — |
| [Jitsi Meet](https://github.com/jitsi/jitsi-meet) | Video calls | Bare | 0.85.2 | — | ✅ | 30.1k ¹ | Apache-2.0 | [iOS](https://apps.apple.com/app/id1165103905) | [Android](https://play.google.com/store/apps/details?id=org.jitsi.meet) | [Web](https://meet.jit.si) ³ |
| [Karakeep](https://github.com/karakeep-app/karakeep/tree/main/apps/mobile) | Bookmarks | Expo (CNG) | 0.86.3 | 57 | ✅ | 29.9k ¹ | AGPL-3.0 | [iOS](https://apps.apple.com/app/id6479258022) | [Android](https://play.google.com/store/apps/details?id=app.hoarder.hoardermobile) | — |
| [Bluesky](https://github.com/bluesky-social/social-app) | Social | Expo (CNG) | 0.86.3 | 57 | ✅ | 18.3k | MIT | [iOS](https://apps.apple.com/app/id6444370199) | [Android](https://play.google.com/store/apps/details?id=xyz.blueskyweb.app) | [Web](https://bsky.app) |
| [Standard Notes](https://github.com/standardnotes/app/tree/main/packages/mobile) | Notes | Bare | 0.78.1 | — | ✅ | 6.7k ¹ | AGPL-3.0 | [iOS](https://apps.apple.com/app/id1285392450) | [Android](https://play.google.com/store/apps/details?id=com.standardnotes) | — |
| [Uniswap Wallet](https://github.com/Uniswap/interface/tree/main/apps/mobile) | Crypto | Bare + Expo modules | 0.85.3 | 56 | ✅ | 5.5k ¹ | GPL-3.0 | [iOS](https://apps.apple.com/app/id6443944476) | [Android](https://play.google.com/store/apps/details?id=com.uniswap.mobile) | — |
| [New Expensify](https://github.com/Expensify/App) | Fintech | Bare + Expo modules | 0.86.0 | 57 | ✅ | 5.1k | MIT | [iOS](https://apps.apple.com/app/id471713959) | [Android](https://play.google.com/store/apps/details?id=org.me.mobiexpensifyg) | [Web](https://new.expensify.com) |
| [Rainbow](https://github.com/rainbow-me/rainbow) | Crypto | Bare + Expo modules | 0.81.6 | 54 | ❌ | 4.4k | GPL-3.0 | [iOS](https://apps.apple.com/app/id1457119021) | [Android](https://play.google.com/store/apps/details?id=me.rainbow) | — |
| [Artsy (Eigen)](https://github.com/artsy/eigen) | Marketplace | Bare + Expo modules | 0.86.3 | 57 | ✅ | 3.8k | MIT | [iOS](https://apps.apple.com/app/id703796080) | [Android](https://play.google.com/store/apps/details?id=net.artsy.app) | — |
| [BlueWallet](https://github.com/BlueWallet/BlueWallet) | Crypto | Bare | 0.85.3 | — | ✅ | 3.3k | MIT | [iOS](https://apps.apple.com/app/id1376878040) | [Android](https://play.google.com/store/apps/details?id=io.bluewallet.bluewallet) | — |
| [MetaMask](https://github.com/MetaMask/metamask-mobile) | Crypto | Bare + Expo modules | 0.86.3 | 57 | ✅ | 3.1k | Custom ² | [iOS](https://apps.apple.com/app/id1438144202) | [Android](https://play.google.com/store/apps/details?id=io.metamask) | — |
| [Mattermost](https://github.com/mattermost/mattermost-mobile) | Chat | Bare + Expo modules | 0.83.9 | 55 | ✅ | 2.7k | Apache-2.0 | [iOS](https://apps.apple.com/app/id1257222717) | [Android](https://play.google.com/store/apps/details?id=com.mattermost.rn) | — |
| [Rocket.Chat](https://github.com/RocketChat/Rocket.Chat.ReactNative) | Chat | Bare + Expo modules | 0.86.3 | 57 | ✅ | 2.4k | MIT | [iOS](https://apps.apple.com/app/id1148741252) | [Android](https://play.google.com/store/apps/details?id=chat.rocket.android) | — |
| [Ledger Wallet](https://github.com/LedgerHQ/ledger-live/tree/develop/apps/ledger-live-mobile) | Crypto | Bare + Expo modules | 0.81.6 | 54 | ✅ | 0.6k ¹ | MIT | [iOS](https://apps.apple.com/app/id1361671700) | [Android](https://play.google.com/store/apps/details?id=com.ledger.live) | — |

All apps use the Hermes engine.

¹ Monorepo: the star count is for the whole repository, not just the mobile app.
² Not a standard SPDX license. See the repo's `LICENSE` file.
³ The Jitsi web app is React DOM that shares code with the mobile app. It does not use react-native-web.

## 2. Tech stack

| App | Bundler | Platforms | Tree shaking | Package manager / monorepo | Navigation | State / data | E2E tests | OTA updates | React Compiler |
|---|---|---|---|---|---|---|---|---|---|
| Joplin | Metro (native), Webpack (web) | iOS, Android, Web | ❌ | Yarn + Lerna | — | Redux | — | — | ❌ |
| Jitsi Meet | Metro | iOS, Android | ❌ | npm | React Navigation | Redux | — | — | ❌ |
| Karakeep | Metro (Expo) | iOS, Android, macOS (iPad app on Mac) | ❌ | pnpm + Turborepo | Expo Router | Zustand, TanStack Query | — | — | ✅ |
| Bluesky | Metro (Expo) | iOS, Android, Web | ⚠️ Experimental (web only) | pnpm | React Navigation | TanStack Query | — | EAS Update | ✅ |
| Standard Notes | Metro | iOS, Android | ❌ | Yarn + Lerna | — (WebView shell) | — (WebView shell) | Detox | — | ❌ |
| Uniswap Wallet | Metro (Expo) | iOS, Android | ❌ | Bun + Nx | React Navigation | Redux Toolkit, Zustand, TanStack Query | Maestro | — | ❌ |
| New Expensify | Re.Pack (Rspack), Metro fallback | iOS, Android, Web | ✅ (Rspack prod) | npm | React Navigation | Onyx | — | — | ✅ |
| Rainbow | Metro | iOS, Android | ❌ | Yarn | React Navigation | Redux, Zustand, Recoil, TanStack Query | Maestro | — | ❌ |
| Artsy (Eigen) | Metro | iOS, Android | ❌ | Yarn | React Navigation | Relay | Maestro | EAS Update | ❌ |
| BlueWallet | Metro | iOS, Android | ❌ | npm | React Navigation | React Context | Detox | — | ❌ |
| MetaMask | Metro | iOS, Android | ❌ | Yarn | React Navigation | Redux Toolkit, TanStack Query | Detox, Appium | EAS Update | ✅ |
| Mattermost | Metro | iOS, Android | ❌ | npm | Expo Router | WatermelonDB | Detox | — | ❌ |
| Rocket.Chat | Metro | iOS, Android | ❌ | pnpm | React Navigation | Redux, Zustand, WatermelonDB | Maestro | — | ✅ |
| Ledger Wallet | Re.Pack (Rspack) | iOS, Android | ✅ (Rspack prod) | pnpm + Nx | React Navigation | Redux Toolkit, Jotai, TanStack Query | Detox | — | ❌ |

## Reference / sample apps

These apps are not in the stores, but they are useful for learning.

| App | RN | Bundler | Platforms | ⭐ Stars | Notes |
|---|---|---|---|---|---|
| [RNTester](https://github.com/react/react-native/tree/main/packages/rn-tester) | `main` (0.87 nightly) | Metro | iOS, Android | 126.7k ¹ | The React Native core team's catalog of every built-in component and API |
