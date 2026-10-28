# WebView idle task starvation repro

**Test page:** https://cjsan30.github.io/webview-idle-repro/

Since Android WebView 147 (still present in 156), idle tasks stop running while a page is static (no new frames, no input).
`requestIdleCallback` callbacks without a `timeout` do not run until the user touches the screen, and `canvas.toBlob` waits for Chromium's idle fallback timeout (4 s on Android) on every call.
Chrome on the same device and WebView 146 are not affected.

The HTML spec does not guarantee when an idle callback runs, so this is not a spec violation by itself.
It is a regression: on a completely idle page the same callbacks run within ~50 ms in WebView 146 and in Chrome 156.

## How to test

1. Open the test page **inside an app that uses Android WebView** (for example an in-app browser of a messenger or social app). Opening it in Chrome gives the normal baseline.
2. Set the screen timeout to 2 minutes or more.
3. Tap **시작 (Start)**, put the phone down and do not touch it for up to 2 minutes. The page deliberately does not update the screen while measuring.
4. Copy the result text with **결과 복사 (Copy result)**.

The page measures:

| Check | Normal | Affected |
| --- | --- | --- |
| `requestIdleCallback(cb, { timeout: 1000 })` ×3 | runs in a few ms to ~50 ms | every call runs only at the 1000 ms timeout (`didTimeout: true`) |
| `requestIdleCallback(cb)` without timeout | ~30 ms | does not run within 30 s |
| 20 chained idle callbacks | all 20 in ~1 s | 0 of 20 within 45 s |
| `canvas.toBlob(cb, 'image/jpeg')` ×3 | ~40 ms each | ~4,020 ms each |

After the result is drawn, the page checks what releases the still-pending idle callback.

## Results so far

### Real device: Samsung Galaxy S25 Ultra (SM-S938N), Android 16

| Engine | Result | rIC timeout 1000 ×3 | rIC no timeout | 20 chained | toBlob ×3 |
| --- | --- | --- | --- | --- | --- |
| WebView 156.0.8078.47 (KakaoTalk 26.6.1 in-app browser) | **reproduced** | 1005 / 1001 / 1001 ms, all `didTimeout` | not run within 30 s | 0 / 20 within 45 s | 4075 / 4058 / 4052 ms |
| Chrome 156 | normal | 0 / 7 / 50 ms | 51 ms | 20 in 957 ms | 59 / 21 / 17 ms |

In the WebView run, the idle callback left pending when the result was drawn stayed pending for **192.8 s**, and ran 31 ms after the first touch.
Drawing the result frame did not release it.

Earlier, with WebView 155.0.8059.43 in a test app, the `toBlob` stall reproduced on every run; Chrome 155.0.8059.84 on the same phone was normal.

### Android Emulator (Pixel 8 image, Android 16, x86_64)

Same page, app restarted before each run:

| WebView | Runs reproduced | Pending callback released |
| --- | --- | --- |
| 146.0.7680.177 | 0 / 3 | — |
| 155.0.8059.43 | 4 / 6 | 11–32 ms after the next frame, without any touch |

On the emulator the stall is intermittent and a single new frame released the pending callback; on the real device it reproduced on every run and only input released it.

An earlier bisect on the emulator with `canvas.toBlob` found the first affected milestone:

| WebView | 133 | 145 | 146 | 147 | 148 | 149 | 150 | 154 | 155 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| stalled runs | 0/10 | 0/8 | 0/20 | 7/20 | 2/8 | 3/8 | 3/5 | 6/8 | 7/8 |

On WebView 147 the stall disappeared with `--disable-features=MainIdleBypassScheduler` (0/20) and returned with the feature enabled.
That feature was enabled by default in Chromium commit `b25cdb892e19` (M147, `refs/heads/main@{#1596486}`, bug 454038686), whose message says the experiment increased the stability of the `requestIdleCallback` API.
It moves the "begin main frame not expected soon" decision, which starts long idle periods, onto the main thread. With WebView's synchronous compositor, long idle periods then appear not to start while nothing is drawn.
The old feature declaration was removed later (`refs/heads/main@{#1604232}`), so the previous behavior can no longer be restored with a flag.

## Workaround

- Pass a `timeout` to `requestIdleCallback` when the work matters.
- Use `canvas.toDataURL()` (synchronous) instead of `canvas.toBlob()` for encoding the user is waiting for. In one app this cut a 24-image PDF export from 261.7 s to 5.8 s on an affected phone.

---

## 한국어 요약

Android WebView 147 이상(156까지 확인)에서는 화면이 변하지 않고 입력도 없으면 idle 작업이 실행되지 않습니다.
- timeout 없는 `requestIdleCallback`은 사용자가 화면을 건드릴 때까지 실행되지 않습니다. 실기기에서 192.8초 동안 실행되지 않았습니다.
- `canvas.toBlob`은 호출할 때마다 약 4초씩 기다립니다.
- 같은 기기의 Chrome 앱과 WebView 146 이하는 정상입니다.
- 표준은 idle 콜백의 실행 시점을 보장하지 않으므로 표준 위반은 아닙니다. 이전 버전과 Chrome에서는 즉시 실행되던 것이 147부터 바뀐 퇴행입니다.

**테스트 방법**
1. 카카오톡, 네이버, 인스타그램 같은 앱 안에서 위 링크를 엽니다.
2. **시작**을 누르고 휴대폰을 내려놓은 뒤 최대 2분간 건드리지 않습니다.
3. 결과를 복사해 공유합니다.

이 페이지는 측정 결과를 어디에도 전송하지 않습니다. 모든 측정은 기기 안에서만 이루어집니다.
