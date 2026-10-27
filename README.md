# WebView idle task starvation repro

**Test page:** https://cjsan30.github.io/webview-idle-repro/

Since Android WebView 147, idle tasks can stop running while a page is static (no new frames).
`requestIdleCallback` callbacks without a `timeout` do not run until the next frame is drawn, and `canvas.toBlob` waits for Chromium's idle fallback timeout (4 s on Android) on every call.
Chrome itself and WebView 146 are not affected.

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

Android Emulator (Pixel 8 image, Android 16, x86_64), same page, app restarted before each run:

| WebView | Runs reproduced | Pending callback released |
| --- | --- | --- |
| 146.0.7680.177 | 0 / 3 | — |
| 155.0.8059.43 | 4 / 6 | 11–32 ms after the next frame, without any touch |

On a Samsung Galaxy S25 Ultra (Android 16, WebView 155.0.8059.43) the stall reproduced on every run; Chrome 155.0.8059.84 on the same phone was normal.

An earlier bisect on the emulator with `canvas.toBlob` found the first affected milestone:

| WebView | 133 | 145 | 146 | 147 | 148 | 149 | 150 | 154 | 155 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| stalled runs | 0/10 | 0/8 | 0/20 | 7/20 | 2/8 | 3/8 | 3/5 | 6/8 | 7/8 |

On WebView 147 the stall disappeared with `--disable-features=MainIdleBypassScheduler` (0/20) and returned with the feature enabled.
That feature was enabled by default in Chromium commit `b25cdb892e19` (M147). It moves the "begin main frame not expected soon" decision, which starts long idle periods, onto the main thread. With WebView's synchronous compositor, long idle periods then appear not to start while nothing is drawn.

## Workaround

- Pass a `timeout` to `requestIdleCallback` when the work matters.
- Use `canvas.toDataURL()` (synchronous) instead of `canvas.toBlob()` for encoding the user is waiting for. In one app this cut a 24-image PDF export from 261.7 s to 5.8 s on an affected phone.

---

## 한국어 요약

Android WebView 147부터 화면이 멈춰 있으면 idle 작업이 실행되지 않습니다.
- timeout 없는 `requestIdleCallback`은 다음 화면이 그려질 때까지 실행되지 않습니다.
- `canvas.toBlob`은 호출할 때마다 약 4초씩 기다립니다.
- Chrome 앱과 WebView 146 이하는 정상입니다.

**테스트 방법**
1. 카카오톡, 네이버, 인스타그램 같은 앱 안에서 위 링크를 엽니다.
2. **시작**을 누르고 휴대폰을 내려놓은 뒤 최대 2분간 건드리지 않습니다.
3. 결과를 복사해 공유합니다.

이 페이지는 측정 결과를 어디에도 전송하지 않습니다. 모든 측정은 기기 안에서만 이루어집니다.
