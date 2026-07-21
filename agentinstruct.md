termupsy: Core System Analysis
What this repo actually is
termupsy is a fork of Termux — the Android terminal emulator / Linux
userland launcher (termux/termux-app upstream). Right now it is
essentially unmodified upstream Termux: the README.md has been retitled
to "TerminOps (Termux for PSYAI)" but the body text, badges, and links
are all still stock Termux. Worth noting since it's a small but real
symptom of the confusion you flagged: the repo is named termupsy, the
zip was termupsy-master, and the README calls the app "TerminOps" —
three different names for the same project, none of them final. Pick one
before you go much further; it'll stop tripping you up in commit messages,
package IDs, and your own notes.
One thing worth knowing before you plan the redesign: there is already a
roadmap doc sitting in this repo at docs/en/wayland-win11-phase1-phase2-roadmap.md.
It outlines a phased plan to turn this into a "universal terminal space"
across Android, Windows 11 IoT, Debian Linux, and RK3399 Chromebooks —
i.e. it's already scoped around the exact heterogeneous-hardware vision
your other projects point at. Whether you wrote it or an earlier session
did, it's a reasonable starting skeleton for the redesign's RFC/scope
section rather than something to rediscover from scratch.
Licensing — read this before you decide how much to keep
This matters more than usual given PSYAI's IP-licensing business model, so
the facts, plainly (I'm not a lawyer — worth a real legal read if license
mixing affects your plans):
Path
License
app/ (everything under it)
GPLv3-only
terminal-emulator/, terminal-view/
Apache 2.0 (originally Jack Palevich's Android-Terminal-Emulator)
termux-shared/ — general
MIT
termux-shared/.../termux/* (the Termux-specific subpackage)
GPLv3-only — except TermuxConstants.java and settings/properties/TermuxPropertyConstants.java, which are carved out as MIT
termux-shared/.../file/filesystem/* (POSIX file-attribute code, from Android's ojluni)
GPLv2-only + Classpath exception
termux-shared/.../shell/StreamGobbler.java
Apache 2.0 (from libsuperuser)
The practical upshot: the single most valuable, most reusable piece of
this codebase — the actual terminal emulation engine (terminal-emulator)
and its Android render layer (terminal-view) — is under the permissive
Apache 2.0 license, not GPL. The copyleft (GPLv3) obligations sit mainly
in app/ and the termux/ subpackage of termux-shared. That's a
meaningfully different position than "the whole thing is GPL," but it's
also not a clean split — GPLv3 code and MIT/Apache code both import from
each other within termux-shared, so "which license actually governs my
redesign" isn't a question I can safely answer for you from a repo read.
Designing the interface layer to stay non-copyleft
Given you're fine open-sourcing termupsy itself under GPL, the follow-up
question — how to keep other projects that talk to it from being pulled
into copyleft — has a well-established answer in FOSS practice: GPL's
copyleft is triggered by linking/combining code into one compiled work,
not generally by two separate programs communicating over a socket,
pipe, or documented API. That's the same reasoning that lets arbitrary
userspace software run on the GPLv2 Linux kernel without itself becoming
GPL (Linus's syscall exception exists to make this explicit), and it's
part of why AGPL exists at all — plain GPL doesn't reach code that
merely talks to a GPL'd service over a network.
Two things worth knowing, both grounded in what's actually in this repo:
You already have a clean license boundary to build the protocol on.
Checking the license-file exceptions against actual package paths: the
generic "run a command, get a structured result back, over a
peer-authenticated local socket" machinery is already MIT, not GPL.
Only the Termux-branded glue wiring it into this specific app is GPLv3.
Already MIT (build the protocol here)
Already GPLv3 (app-specific glue)
shell/command/ExecutionCommand.java
termux/plugins/TermuxPluginUtils.java
shell/command/runner/app/AppShell.java
termux/shell/am/TermuxAmSocketServer.java
shell/command/result/{ResultData,ResultConfig,ResultSender}.java
termux/shell/TermuxShellManager.java
shell/command/environment/*
termux/shell/command/runner/terminal/TermuxSession.java
shell/am/{AmSocketServer,AmSocketServerRunConfig}.java
all of app/ (the Android application itself)
net/socket/local/* (peer-credential-checked Unix socket server/client)

If the new mesh protocol is built as an extension of the left column
rather than on top of RunCommandService/TermuxPluginUtils, a client
that only speaks the protocol never has to touch GPL'd code at all.
This reverses one thing I said in the first pass. I'd suggested
collapsing termux-shared's generic-vs-termux/-specific split once you
weren't maintaining a library for six companion apps. Don't — sharpen
it instead. Generic/MIT holds the protocol: message schema, transport,
session/capability negotiation, the result model. termux/-or-app/
GPL holds only the policy glue that's genuinely specific to this terminal
app (notifications, UI, the manifest surface). New mesh-facing types
default to the MIT side; they only move to the GPL side if they're doing
something intrinsically Termux-app-specific.
Concretely:
Don't build the new protocol on RunCommandService/TermuxPluginUtils — they're Android-Intent-shaped anyway, which doesn't fit an ESP32-P4 or a Windows 11 IoT node regardless of license.
Give the protocol its own explicit license: its own module, its own LICENSE.md, MIT or Apache 2.0 — rather than an exception buried inside someone else's license file. That's literally how TermuxConstants.java/TermuxPropertyConstants.java are handled today, and it took reading the license file line by line to find it. Make the boundary obvious instead.
Build the transport on net/socket/local/* — already MIT, already has the peer-credential auth a mesh needs anyway.
Keep app/'s job narrow: it's a GPL'd application that happens to implement the protocol, not the protocol's definition.
One caveat, stated once: this is well-established FOSS practice and the
FSF's own published position on communication-vs-linking, not case law
tested on your exact architecture — the line between "separate program"
and "combined work" gets fuzzier the more tightly two things are coupled
at runtime. Given PSYAI's model is IP licensing, worth having actual
counsel sanity-check the final module boundary before it's represented as
a guarantee to anyone connecting to termupsy.
Module map
Code
settings.gradle only declares 4 modules (app, termux-shared,
terminal-emulator, terminal-view) — everything else is non-code
(images, F-Droid/Play listing text, docs, the Gradle wrapper). Those
non-code dirs aren't causing your confusion; they're already cleanly
separated from the source tree. The confusion is inside the 4 modules,
specifically inside termux-shared.
The actual core — preserve this wholesale
terminal-emulator (8,535 lines) — this is the crown jewel and the
part you should touch the least. TerminalEmulator.java (2,617 lines) is
the full ANSI/xterm escape-sequence state machine; TerminalBuffer,
TerminalRow, WcWidth (Unicode width table), KeyHandler, and
TerminalSession round out the model. src/main/jni/termux.c is the
native PTY fork/exec layer — this is literally where a shell process gets
spawned. There are ~2,000 lines of unit tests here (TerminalTest,
CursorAndScreenTest, ResizeTest, KeyHandlerTest, etc.) covering
escape-sequence parsing, cursor movement, resize behavior — genuinely
valuable regression coverage if you're going to modify emulator behavior.
terminal-view (2,833 lines) — TerminalView.java (1,500 lines) is
the View that draws the buffer and handles touch/key input;
TerminalRenderer does the actual glyph drawing; the textselection/
package handles long-press selection handles; GestureAndScaleRecognizer
handles pinch-zoom and fling. All core UX, all worth keeping.
app/ core files (by line count):
TermuxActivity.java (1,013) — the terminal screen itself
TermuxService.java (959) — foreground service, owns session lifecycle
terminal/TermuxTerminalViewClient.java (802) — bridges terminal-view ↔ app (key handling, bell, theming)
terminal/TermuxTerminalSessionActivityClient.java (528) — session lifecycle callbacks
TermuxInstaller.java (386) — extracts the bootstrap userland zip on first run
TermuxApplication.java, terminal/TermuxActivityRootView.java, terminal/TermuxSessionsListViewController.java, terminal/io/* (extra-keys row, toolbar paging) — all core UX plumbing
termux-shared core: shell/command/ (ExecutionCommand — the model
every command execution path uses; environment/* builds PATH/HOME/PREFIX
env vars; runner/app/AppShell.java runs non-interactive background
commands — used by TermuxService itself, not just the plugin path, so
don't treat it as plugin-only), file/* (POSIX-grade file utilities),
settings/preferences + settings/properties (generic property-file
engine backing termux.properties), logger/, notification/, crash/,
android/, theme/.
Already stripped (this pass — see termupsy-core.zip)
I removed the one cluster I could trace completely and safely: settings
UI for four companion apps that aren't in this repo (Termux:API,
Termux:Float, Termux:Tasker, Termux:Widget — each is a separate GitHub
repo/APK; none of their code lives here). 26 files gone, 4 files edited.
Full itemized list in STRIPDOWN_NOTES.md inside the zip. I traced every
reference by hand and re-grepped afterward to confirm nothing dangling
remains — but I don't have Android SDK/NDK or network access in this
sandbox, so this has not been compiled. Treat it as a careful
first pass, not a verified one; do a real build before trusting it.
I deliberately left TermuxConstants.java/TermuxPreferenceConstants.java's
per-companion-app nested classes alone, even though they only exist for
those same absent apps — pulling them out cascades into TermuxUtils.java
(About-screen / issue-link builder) and TermuxActivity.java:739
(a "jump to Termux:Styling" menu action), which is a smaller, safer
follow-up once you're ready for it.
Peripheral, but legitimate — not "cruft," just not core
These integrate the app with the rest of Android and have nothing to do
with the companion-app ecosystem. I'd keep them unless you have a
specific reason not to:
FileReceiverActivity + TermuxOpenReceiver + TermuxDocumentsProvider — let other apps share files into Termux, let Termux open/share files out, and let a file manager browse $HOME via Storage Access Framework.
net/uri, net/url — small URI/URL parsing helpers.
Decision points — these need your call, not mine
I traced each of these enough to know they're not simple deletions —
each reaches into core session/command code in ways that would need real
design decisions, not just deletion:
RUN_COMMAND / plugin automation (RunCommandService,
TermuxPluginUtils, ResultData/ResultConfig/ResultSender) — the
mechanism that lets an external Android app (originally Termux:Tasker)
tell Termux to run a command and get structured results back. It's
wired into TermuxService.java, ExecutionCommand.java, and
TermuxSession.java — all core files. This is worth pausing on rather
than reflexively cutting: the underlying idea (something external can
command this terminal and get a result back) is close to what a
hardware-mesh control plane needs. The current implementation is just
very Android-Intent/Tasker-shaped, not mesh-shaped. Worth deciding:
kill it, keep it as-is, or mark it as the seed of a protocol you'll
redesign.
The local socket "am" server (net/socket/local/* +
shell/am/*, notably TermuxAmSocketServer.java) — a Unix-domain-socket
server with peer-credential checking, currently used only to proxy
Android's am (Activity Manager) command-line tool from inside the
sandboxed shell. Structurally, this is a real local-IPC primitive with
authentication already built in — potentially a useful foundation for
inter-node communication in your mesh, or dead weight if you don't need
am proxying specifically. Worth a look before deciding either way.
Bootstrap/rootfs source. app/build.gradle has a downloadBootstraps
task that fetches prebuilt userland archives per architecture from
github.com/termux/termux-packages releases at build time — this is
where the actual Debian/Ubuntu-ish userland content comes from; none of
it lives in this repo. Given your Straeus/debootstrap work, this is
probably the first thing you replace rather than the last — worth
deciding early since it shapes TermuxInstaller.java and the
environment-variable setup in termux-shared too.
sharedUserId in AndroidManifest.xml + the 6 companion-app name
placeholders in app/build.gradle. This is what lets Termux and its
companion APKs share a Linux UID on-device — a mechanism Google already
discourages for modern targetSdk. Only worth keeping if you actually
plan a multi-APK companion ecosystem; nothing in what you've described
suggests you do.
Smaller things worth cleaning up as you go
termux-shared has two parallel hierarchies: generic reusable code
(android/, file/, shell/, settings/, net/, logger/, etc.) and
a termux/ subpackage that specializes it for this specific app family.
It originally existed because termux-shared was published to JitPack
(jitpack.yml at repo root) for six separate companion-app repos to
consume. Update given the licensing conversation below: don't collapse
this split — it also happens to be your MIT/GPL boundary. Sharpen it
instead. See "Designing the interface layer to stay non-copyleft."
Naming collision: termux-shared has both an activities/ package
(actual Activity subclasses — ReportActivity, TextIOActivity) and an
activity/ package (utility classes for working with activities —
ActivityUtils, ActivityErrno). Same word, different meaning, one
letter apart. Worth a rename regardless of anything else you do.
terminal-view/support/PopupWindowCompatGingerbread.java is a compat
shim for Android 2.3 (API 9–10); minSdkVersion here is 21, so the
branch that uses it can never fire. Still referenced from
TextSelectionHandleView.java, so it's not an orphaned file, just a
dead branch — low priority, but flag it as vestigial.
app/build.gradle still supports an apt-android-5 bootstrap variant
for Android 5/6, which the README itself says is already deprecated.
Build-time app names are declared twice, independently: once as
manifestPlaceholders in app/build.gradle (feeds AndroidManifest.xml),
and again as hardcoded DOCTYPE ENTITY declarations at the top of
strings.xml (feeds that file only). They happen to agree today; they
have no mechanism keeping them in sync.
Suggested path forward
Settle the name (termupsy / TerminOps / something else) and use it
everywhere — package ID, repo, README — before writing new code.
Decide the bootstrap/rootfs question (#3 above) first; it's upstream of
TermuxInstaller and the environment setup, so it determines how much
of termux-shared's environment code survives as-is.
Decide the RUN_COMMAND/automation question (#1) next, since it's the
one place where "delete" and "redesign into something mesh-shaped"
genuinely diverge — worth deciding before touching TermuxService.java.
Once those two are settled, the companion-app constant cleanup
(TermuxConstants/TermuxPreferenceConstants/TermuxUtils/
TermuxActivity:739) becomes a quick, low-risk pass — happy to do that
one next.
Get a real Gradle build running (Android SDK + NDK) before any further
structural changes, so we have actual compiler feedback instead of
hand-traced greps.
Everything above is grounded in reading the actual code paths, not
assumptions from the repo layout — file paths and line references are
exact as of this pass.
