# AWS Device Farm  Test Android/iOS/Web Apps on Real Devices

## Purpose

AWS Device Farm is an **app testing service** that lets you **run automated tests and manually use your Android, iOS, or web app on real, physical phones & tablets (and desktop browsers) hosted by AWS**  no device lab to buy or maintain. You get parallel test execution, video/log/screenshot reports, and **remote access** (interact via a browser or an **Appium endpoint**) to reproduce bugs and check visuals. (Only available in **us-west-2  Oregon**.) Pay **$0.17/device-minute** (1,000 free trial minutes) or unlimited metered plans.

## Main use cases

- **Automated app testing**  run Appium/JUnit/Espresso/XCTest/XCUITest suites (or built-in "Fuzz", "Compatibility", "Performance" tests) across many devices **in parallel**
- **Web-app testing on real mobile devices** + desktop browsers (Selenium)  cross-device/browser regression
- **Manual QA / bug reproduction**  remotely drive a real device (swipe/tap/rotate, mock location, network shaping, orientation, screenshot/video/logs)
- **Appium development/debugging**  connect a **managed Appium endpoint** to write/debug scripts with live logs & video
- **Quality gates in CI/CD**  Device Farm integrates with CodePipeline/CodeBuild & Jenkins-style flows so every build tests against a real device fleet

## Key features

- **Real devices**  physical iOS/Android (non-emulator) hosted at AWS web/desktop-browser testing too
- **Parallel runs**  test apps on **multiple devices simultaneously** pool/no-choice device selection & reports
- **Remote access**  interactive session via browser with video/activity logs, Appium endpoint, **location mocking, network profiles, orientation, screenshots**
- **Built-in tests**  Fuzz, Compatibility/Performance, no-code smoke on upload
- **Artifacts & reports**  high-level results, low-level logs, screenshots, video **CI/CD** integration (CodePipeline, Jenkins plugin, CLI/SDK)
- **Pricing flexibility**  metered ($0.17/dev-min) vs **unmetered** (flat monthly per device), private device subscriptions

## When to use

- Need **real-device compatibility/regression testing** without buying a device farm
- Reproduce support issues on unfamiliar devices/OS versions quickly (remote access)
- Scale automated mobile/web test suites across many devices in parallel for **CI quality gates**
- Validate localization (real-world network/location profiles), performance hints, and app-install/upgrade flows

## Important limitation

- **Cost/throughput**: per **device-minute** pricing (or flat/unmetered plans) can grow with heavy parallel fleets **1,000-minute free trial** only. **US-West-2 (Oregon) only** no arbitrary device models (must exist in Device Farm's catalog), and **you can't access the physical OS/root shell**  it's device-under-test, not a sandbox. Simultaneous connection limited by purchased **device slots** long/interactive human sessions are billed per minute. Complex private labs with total control → build your own (e.g., open-source + streaming) or use **private device** subscriptions.

## SAA relevance

- "**Test mobile/web apps on real devices**, remote access, parallel automated runs" → **AWS Device Farm**
- "**CI/CD dev+test quality gate** for mobile apps on AWS" → Device Farm (integrate with CodePipeline/Build)
- "Performance & **cross-browser/device testing**" → Device Farm (real devices, browser support)
- "**Reproduce a bug on a specific device/OS**" → Device Farm remote-access sessions
- Exam traps: **Device Farm is for testing your apps**, not for scale-out compute/App hosting (that's EC2/Amplify/App Runner) remember the **Oregon-only** constraint and **device-minute** billing don't confuse with **AWS App Test**(none) or Lambda (execution)  it's a QA/manual device environment.