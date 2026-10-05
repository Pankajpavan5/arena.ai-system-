# Android APK Decompilation Guide — Low-Resource Linux

> Practical workflow for downloading, inspecting, decompiling, troubleshooting, and analyzing Android APKs with **2 GB RAM, 2 CPU threads, and 20 GB storage**.
>
> **Scope:** authorized analysis, interoperability, security research, and learning. Do not use this workflow to bypass licensing, access controls, DRM, or other protections.

## 1. Target APK

Example target: **Xiaomi Joyose 2.5.22**, package `com.xiaomi.joyose`.

APKMirror currently lists the 2.5.22 build as **arm64-v8a, Android 7.0+, nodpi, 2.23 MB**, uploaded September 7, 2026. APKMirror also lists its SHA-256 certificate fingerprint as:

```text
c9009d01ebf9f5d0302bc71b2fe9aa9a47a432bba17308a3111b75d7b2149025
```

Use the exact APKMirror download page supplied for this build rather than guessing a direct file URL.

## 2. Download from APKMirror

### Browser method

Open:

```text
https://www.apkmirror.com/apk/xiaomi-inc/joyose/xiaomi-joyose-2-5-22-release/xiaomi-joyose-2-5-22-android-apk-download/download/?key=19cbd38635fbe6640952484c3a989d4f8b4a5f88
```

Complete APKMirror's download flow and save the APK as:

```text
joyose-2.5.22.apk
```

### Linux verification

After downloading:

```bash
ls -lh joyose-2.5.22.apk
file joyose-2.5.22.apk
sha256sum joyose-2.5.22.apk
```

Record the output in `checksums.txt`.

**Important:** APKMirror's page certificate information is not the same thing as the APK's complete file SHA-256. Keep both records separately.

## 3. Prepare the 2 GB RAM / 2-thread machine

Do NOT blindly "optimize" the entire OS. The goal is to reduce memory pressure and temporary-file consumption without destabilizing Linux.

Check resources:

```bash
free -h
nproc
df -h
java -version
```

Create a dedicated workspace:

```bash
mkdir -p ~/apk-work/{input,jadx-out,apktool-out,smali,logs,tmp}
cd ~/apk-work
```

Keep at least **5–8 GB free** before starting a large decompilation. APKs can expand substantially during extraction/decompilation.

### Enable/check swap

```bash
swapon --show
free -h
```

If your Linux environment has no swap and you administer the machine, configure a swapfile using your distribution's documented procedure. Do not create a swapfile on a nearly-full disk.

### Reduce background load

Before a large JADX run:

```bash
ps aux --sort=-%mem | head -15
```

Stop only applications you recognize and do not need. Avoid killing system services just to gain RAM.

## 4. Install Java

JADX requires a 64-bit Java runtime; current upstream documentation says Java 11+ and current source builds require JDK 17+.

For Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk unzip wget curl git
```

Verify:

```bash
java -version
javac -version
```

If Java is installed somewhere non-standard:

```bash
readlink -f "$(command -v java)"
echo "$JAVA_HOME"
```

Set `JAVA_HOME` if required:

```bash
export JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v java)")")")"
export PATH="$JAVA_HOME/bin:$PATH"
```

## 5. Install JADX

### Preferred: official release

Download the latest stable JADX release from its official GitHub releases page:

```text
https://github.com/skylot/jadx/releases
```

Then:

```bash
mkdir -p ~/tools
cd ~/tools

# Replace <VERSION> with the release you downloaded.
unzip jadx-<VERSION>.zip
cd jadx-<VERSION>
chmod +x bin/jadx bin/jadx-gui

./bin/jadx --version
```

For a headless 2 GB RAM server, use **`jadx`**, not `jadx-gui`.

### Build from source

Only do this if necessary:

```bash
git clone https://github.com/skylot/jadx.git
cd jadx
./gradlew dist
```

This itself consumes additional CPU/RAM, so a prebuilt release is preferable on a 2 GB machine.

## 6. First-pass APK inspection

Before decompiling:

```bash
cd ~/apk-work

unzip -l input/joyose-2.5.22.apk | less
```

Useful checks:

```bash
unzip -l input/joyose-2.5.22.apk | grep -E 'classes.*\.dex$'
unzip -l input/joyose-2.5.22.apk | grep -E 'lib/.+\.so$'
unzip -l input/joyose-2.5.22.apk | grep -E 'resources\.arsc|AndroidManifest\.xml'
```

Extract a copy without modifying the original:

```bash
mkdir -p raw
unzip -q input/joyose-2.5.22.apk -d raw/
```

This lets you determine whether the APK contains multiple DEX files or native libraries before spending resources on full decompilation.

## 7. JADX — low-memory configuration

Create a launcher:

```bash
cat > ~/apk-work/run-jadx.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

APK="${1:?Usage: $0 input.apk output-dir}"
OUT="${2:?Usage: $0 input.apk output-dir}"

mkdir -p "$OUT"

exec "$HOME/tools/jadx-<VERSION>/bin/jadx" \
  --threads-count 2 \
  --no-res \
  --show-bad-code \
  --output-dir "$OUT" \
  "$APK"
EOF

chmod +x ~/apk-work/run-jadx.sh
```

Replace `<VERSION>` with your actual installed version.

### Why `--no-res`?

For a first Java-focused pass, skipping resource decoding can reduce work. It does **not** mean resources are lost from the original APK; the untouched APK remains available.

If you need resources later, run a separate resource-analysis pass.

### Run

```bash
cd ~/apk-work

./run-jadx.sh \
  input/joyose-2.5.22.apk \
  jadx-out/joyose
```

Log the run:

```bash
./run-jadx.sh input/joyose-2.5.22.apk jadx-out/joyose \
  2>&1 | tee logs/jadx.log
```

## 8. If JADX fails or produces incomplete Java

This is normal. JADX explicitly warns that it cannot decompile 100% of every APK.

Do **not** repeatedly increase memory until the machine starts swapping uncontrollably.

First inspect:

```bash
grep -Ei 'error|exception|failed|warn' logs/jadx.log | head -100
```

Then check:

```bash
find jadx-out/joyose -type f | wc -l
du -sh jadx-out/joyose
free -h
df -h .
```

### Recovery strategy

Use a staged pipeline:

```text
APK
 ├── ZIP inspection
 ├── DEX inventory
 │    ├── classes.dex
 │    ├── classes2.dex
 │    └── ...
 ├── JADX → Java approximation
 ├── Apktool → resources + smali
 └── native .so → separate native-analysis workflow
```

If JADX struggles with the complete APK, work on individual DEX files.

Extract DEX:

```bash
mkdir -p dex
unzip -j input/joyose-2.5.22.apk 'classes*.dex' -d dex/
```

Then decompile one file at a time:

```bash
mkdir -p jadx-out/classes1

"$HOME/tools/jadx-<VERSION>/bin/jadx" \
  --threads-count 2 \
  --no-res \
  --show-bad-code \
  -d jadx-out/classes1 \
  dex/classes.dex
```

Repeat for additional DEX files.

This reduces peak memory compared with processing a very large multi-DEX APK as one job.

## 9. Apktool — resources + Smali

JADX produces Java-like source. Apktool is useful for the lower-level representation: resources, manifest decoding, and Smali.

Install/use the official Apktool release:

```text
https://github.com/iBotPeaches/Apktool/releases
```

Then:

```bash
apktool d input/joyose-2.5.22.apk \
  -o apktool-out/joyose \
  --use-aapt2
```

If you only need code investigation, prioritize Smali and avoid unnecessary rebuild operations.

## 10. Understanding "obfuscated" vs "broken" code

These are different problems.

### Obfuscation

Examples:

```text
a.a.a()
b.c()
x.y.z()
```

Names may have been deliberately shortened.

JADX's deobfuscation features can improve readability, but **they cannot reconstruct the original developer names with certainty**.

### Decompiler failure

Examples:

```text
/* JADX WARN: ... */
throw new UnsupportedOperationException(...)
```

or malformed/partial Java.

This does not necessarily mean the APK is corrupted. It can mean the DEX contains bytecode patterns that JADX cannot cleanly reconstruct into Java.

### Native code

If you find:

```text
lib/arm64-v8a/*.so
```

JADX will not turn native machine code into Java. Analyze native libraries separately with appropriate tools.

## 11. "Autofix" pipeline

There is no universal command that can automatically repair every decompiler error.

Use an evidence-driven loop:

```bash
# 1. Preserve original
cp input/joyose-2.5.22.apk input/original.apk

# 2. Collect logs
grep -Ei 'error|exception|failed|warn' logs/jadx.log > logs/jadx-errors.txt || true

# 3. Identify affected classes
grep -E 'Class|Method|Exception' logs/jadx-errors.txt | head -100

# 4. Compare Java against Smali
# JADX output: jadx-out/joyose/
# Apktool output: apktool-out/joyose/smali*/
```

For a suspicious class:

```text
DEX bytecode
   ↓
Smali
   ↓
JADX Java approximation
   ↓
Compare control flow
   ↓
Manually correct interpretation
```

Do not automatically rewrite every suspicious construct. A script that blindly replaces `null`, casts, branches, or exception handlers can create code that looks valid but no longer represents the APK's actual behavior.

## 12. Safe automation pattern

A useful automation script should **report** problems first:

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="${1:-jadx-out/joyose}"
REPORT="${2:-logs/autofix-report.txt}"

{
  echo "=== JADX analysis report ==="
  date
  echo

  echo "[Warnings]"
  grep -RniE 'JADX WARN|WARN:|ERROR:' "$ROOT" 2>/dev/null | head -200 || true

  echo
  echo "[Potentially incomplete Java]"
  grep -RniE 'UnsupportedOperationException|JADX WARN|/* ERROR */' \
    "$ROOT" 2>/dev/null | head -200 || true
} > "$REPORT"

echo "Report written to $REPORT"
```

Then manually inspect the reported classes against Smali.

## 13. Very large APKs

For APKs much larger than this Joyose example:

### Do not

```text
❌ unzip everything repeatedly
❌ run JADX GUI on a 2 GB RAM machine
❌ run multiple decompilers simultaneously
❌ keep several full extracted copies
❌ blindly raise Java heap until swap is exhausted
```

### Prefer

```text
✅ keep one immutable original APK
✅ inventory ZIP contents first
✅ process DEX files in stages
✅ use 1–2 worker threads
✅ keep output on fast local storage
✅ remove temporary files after validation
✅ use Smali when Java reconstruction fails
```

Storage check:

```bash
du -sh ~/apk-work/*
df -h ~/apk-work
```

Clean only disposable temporary output:

```bash
rm -rf ~/apk-work/tmp/*
```

Never delete the original APK or verified evidence accidentally.

## 14. Rebuild warning

Decompilation is not the same as successful recompilation.

A JADX-generated Java tree is primarily for **reading and analysis**. If the goal is modification/rebuild, work from the original APK with Apktool/Smali or the appropriate source-level project.

After rebuilding, Android package signing changes unless the original signing key is available. A rebuilt APK therefore generally cannot be treated as the original signed package.

## 15. Minimal command checklist

```bash
# Inspect system
free -h
nproc
df -h
java -version

# Verify APK
sha256sum input/joyose-2.5.22.apk
unzip -l input/joyose-2.5.22.apk | grep -E 'classes.*\.dex$'

# Extract DEX inventory
mkdir -p dex
unzip -j input/joyose-2.5.22.apk 'classes*.dex' -d dex/

# JADX
"$HOME/tools/jadx-<VERSION>/bin/jadx" \
  --threads-count 2 \
  --no-res \
  --show-bad-code \
  -d jadx-out/joyose \
  input/joyose-2.5.22.apk

# Apktool
apktool d input/joyose-2.5.22.apk -o apktool-out/joyose --use-aapt2

# Find warnings
grep -RniE 'JADX WARN|ERROR|Exception' jadx-out/joyose \
  > logs/decompile-warnings.txt || true
```

## 16. Practical expectation for this specific APK

The Joyose 2.5.22 APK listed by APKMirror is only about **2.23 MB**, so a machine with 2 GB RAM and 20 GB storage should not need aggressive system optimization for this particular target.

The low-memory workflow becomes important when the APK contains many DEX files, very large resources, generated code, or large native libraries.

**Rule:** optimize the workflow first, the operating system second.

## References

- APKMirror — Xiaomi Joyose 2.5.22:
  https://www.apkmirror.com/apk/xiaomi-inc/joyose/xiaomi-joyose-2-5-22-release/
- JADX:
  https://github.com/skylot/jadx
- Apktool:
  https://github.com/iBotPeaches/Apktool
